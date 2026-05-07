---
title: "SQL Injection Deep Dive — Parameterized Queries, ORMs, and Second-Order SQLi"
date: 2026-05-07
updated: 2026-05-07
tags: [security, sql-injection, web-security, jdbc, prisma, hibernate, nosql-injection]
---

# SQL Injection Deep Dive — Parameterized Queries, ORMs, and Second-Order SQLi

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `security` `sql-injection` `web-security` `jdbc` `prisma` `hibernate` `nosql-injection`

---

## Table of Contents

- [Summary](#summary)
- [1. Anatomy of an Injection](#1-anatomy-of-an-injection)
- [2. The Real Fix: Parameterized Queries](#2-the-real-fix-parameterized-queries)
- [3. Library by Library — TypeScript and Java](#3-library-by-library--typescript-and-java)
- [4. Where ORMs Stop Protecting You](#4-where-orms-stop-protecting-you)
- [5. Second-Order SQL Injection](#5-second-order-sql-injection)
- [6. Blind SQL Injection — Boolean and Time-Based](#6-blind-sql-injection--boolean-and-time-based)
- [7. NoSQL Injection](#7-nosql-injection)
- [8. Stored Procedures — Help and Limits](#8-stored-procedures--help-and-limits)
- [9. Defense in Depth](#9-defense-in-depth)
- [10. Detection in Production](#10-detection-in-production)
- [11. Notable Incidents and CVEs](#11-notable-incidents-and-cves)
- [Related](#related)
- [References](#references)

---

## Summary

SQL injection is the textbook injection vulnerability — A03:2021 in the OWASP Top 10 — and despite being the oldest serious web bug it still ships in production every year because developers conflate two different things: building a *value* into a query (safe with placeholders) and building the *structure* of a query (never safe with placeholders, and the trap most ORMs leave open). This doc treats SQLi as a problem of *where the SQL parser stops trusting your input*, then walks the defenses across JDBC, node-postgres, Prisma, Sequelize, JPA/Hibernate, and MyBatis. It also covers second-order SQLi (your sanitization runs once, then the value bites you on the second query), blind SQLi (no error output, so attackers exfiltrate one bit at a time), NoSQL operator injection (Mongoose `$where` is still being patched in 2025), and the operational layers — least-privilege DB users, structured query logging, `pg_stat_statements` anomaly review — that catch the bugs your code reviewers missed.

The mental model worth keeping: **a prepared statement sends the query plan and the values to the database in two separate protocol messages.** The database parses placeholders before it ever sees user input, so user input cannot change the plan. String interpolation does the opposite: it ships a single rendered query and asks the parser to figure out what's data and what's syntax. Every SQLi defense is a variation of "don't do that," and every SQLi bug is a variation of "we did it anyway, just hidden behind an abstraction."

---

## 1. Anatomy of an Injection

### 1.1 The classic concatenation bug

```typescript
// Express + node-postgres — the bug that is older than the language it's written in
app.get('/users', async (req, res) => {
  const name = req.query.name;
  const result = await pool.query(
    `SELECT id, email FROM users WHERE name = '${name}'`
  );
  res.json(result.rows);
});
```

Request: `GET /users?name=' OR 1=1 --`. The rendered query becomes:

```sql
SELECT id, email FROM users WHERE name = '' OR 1=1 --'
```

Every row leaks. The parser cannot distinguish between the closing quote you intended and the closing quote the attacker supplied because by the time the bytes reach PostgreSQL, *they are the same string*. The "fix" of escaping quotes (`name.replace("'", "''")`) is a known dead end — Unicode normalization, double-encoded backslashes, and database-specific escape rules have all been used to bypass hand-rolled escaping. The Postgres docs explicitly recommend parameterized queries instead.

### 1.2 Dynamic SQL in a stored procedure

The bug isn't language-specific. Postgres `EXECUTE` and SQL Server `sp_executesql` happily inject too:

```sql
-- PL/pgSQL — same vulnerability, different layer
CREATE FUNCTION find_user(p_name text) RETURNS SETOF users AS $$
BEGIN
  RETURN QUERY EXECUTE 'SELECT * FROM users WHERE name = ''' || p_name || '''';
END;
$$ LANGUAGE plpgsql;
```

Use `EXECUTE ... USING` to bind parameters, or build a static query when the structure is known.

### 1.3 What injection actually grants

Once an attacker can append SQL, the impact ladder is roughly:

1. **Read more rows than intended** — the union/comment trick above.
2. **Read other tables** — `UNION SELECT password_hash, NULL FROM admin_users`.
3. **Write or delete data** — depends on the DB user's privileges (covered in §9).
4. **Read files / execute commands** — Postgres `COPY ... FROM PROGRAM`, MySQL `LOAD_FILE`, MSSQL `xp_cmdshell`, all gated by privilege.
5. **Pivot to RCE** — combined with insecure deserialization or a poorly sandboxed extension.

This is why "just an injection" is misleading; the impact is bounded by the database account, not by the application code.

---

## 2. The Real Fix: Parameterized Queries

### 2.1 Wire-protocol view

A prepared statement is two messages on the PostgreSQL frontend/backend protocol: `Parse` (with `$1`, `$2` placeholders) and `Bind` (with the values). The server compiles a plan from the parse message *before* the values arrive, so the values are always treated as data — they cannot turn into syntax. The same model exists in MySQL (`COM_STMT_PREPARE` / `COM_STMT_EXECUTE`) and SQL Server (sp_prepexec).

This is why "parameterized" beats "escaped": escaping pretends to neutralize syntax inside a single rendered string. Parameterization removes the user input from the parse step entirely.

### 2.2 What this looks like at the call site

```typescript
// node-postgres — placeholders are $1, $2, ...
const result = await pool.query(
  'SELECT id, email FROM users WHERE name = $1',
  [req.query.name]
);
```

```java
// JDBC — placeholders are ?
try (PreparedStatement ps = conn.prepareStatement(
        "SELECT id, email FROM users WHERE name = ?")) {
    ps.setString(1, name);
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) { /* ... */ }
    }
}
```

In both cases, the driver ships the placeholder query and the bound value in separate fields of the wire protocol. There is no string concatenation in your code, and there is no string concatenation in the driver — the value never enters the SQL parser.

### 2.3 Why "?" and "$1" are not template literals

A common confusion: `db.query('SELECT ... WHERE id = ?', [userId])` is *not* `db.query(\`SELECT ... WHERE id = ${userId}\`)` with extra steps. The first sends two protocol messages. The second renders a string and ships it to the parser. Linters such as `eslint-plugin-security`'s `detect-non-literal-fs-filename` and `@typescript-eslint/no-base-to-string` won't catch this; reviewers must.

---

## 3. Library by Library — TypeScript and Java

| Library | Safe pattern | Common footgun |
|---------|--------------|----------------|
| **node-postgres** | `pool.query('... WHERE id = $1', [id])` | `pool.query(\`... WHERE id = ${id}\`)` template strings |
| **Prisma** | `prisma.user.findMany({ where: { name } })` and `prisma.$queryRaw\`SELECT ... WHERE id = ${id}\`` | `prisma.$queryRawUnsafe('... ' + name)` — explicitly named "Unsafe" |
| **Sequelize** | `User.findAll({ where: { name } })`, or `sequelize.query(sql, { replacements })` | String concatenation into `sequelize.query`, or `where: literal('...' + input + '...')` |
| **TypeORM** | `repo.createQueryBuilder().where('id = :id', { id })` | `.where('id = ' + id)` or `Raw(alias => '...' + input)` |
| **Knex** | `knex('users').where({ id })` and `knex.raw('? = ?', [a, b])` | `knex.raw('SELECT ... ' + input)` |
| **JDBC** | `PreparedStatement` with `?` placeholders | `Statement.executeQuery("SELECT ... " + input)` |
| **JPA / Hibernate** | `em.createQuery("FROM User u WHERE u.name = :name").setParameter("name", n)` | `em.createNativeQuery("SELECT ... " + name)` |
| **Spring `JdbcTemplate`** | `jdbc.queryForObject("... WHERE id = ?", args, mapper)` | `jdbc.queryForObject("... WHERE id = '" + id + "'", ...)` |
| **MyBatis** | `#{name}` in mapper XML | `${name}` in mapper XML — substituted as-is |
| **R2DBC** | `connection.createStatement(sql).bind("$1", value)` | Concatenating into the SQL string |

### 3.1 Prisma — the `Unsafe` suffix is a load-bearing word

Prisma's tagged-template `$queryRaw` is safe because Prisma extracts the interpolated values and binds them as parameters. The danger is `$queryRawUnsafe`, which does string concatenation. The naming is intentional — anything ending in `Unsafe` should appear in a code review checklist.

```typescript
// Safe: tagged template, value goes through bind
const users = await prisma.$queryRaw<User[]>`
  SELECT id, email FROM users WHERE name = ${name}
`;

// Unsafe: explicit string, concatenated
const users = await prisma.$queryRawUnsafe(
  `SELECT id, email FROM users WHERE name = '${name}'`
);
```

Prisma's documentation calls out the distinction in the raw-queries reference; treat any `Unsafe` call as a CODEOWNERS-required review.

### 3.2 MyBatis — `#{}` vs `${}`

MyBatis ships two interpolation syntaxes that look almost identical:

```xml
<!-- Safe: #{} compiles to a JDBC PreparedStatement parameter -->
<select id="byName" resultType="User">
  SELECT id, email FROM users WHERE name = #{name}
</select>

<!-- Unsafe: ${} performs raw string substitution before the SQL is parsed -->
<select id="byName" resultType="User">
  SELECT id, email FROM users WHERE name = '${name}'
</select>
```

`${}` exists for cases where you genuinely need to substitute structure (a column name, an `ORDER BY` direction). It must never carry user input directly — see §4.

### 3.3 JPA / Hibernate — JPQL is parameterizable, native is your problem

```java
// JPQL — bound parameters are first-class
TypedQuery<User> q = em.createQuery(
    "SELECT u FROM User u WHERE u.email = :email", User.class);
q.setParameter("email", email);

// Native query — same parameter binding works, but only if you use it
Query n = em.createNativeQuery(
    "SELECT * FROM users WHERE email = ?1", User.class);
n.setParameter(1, email);
```

The trap is `createNativeQuery("SELECT * FROM users WHERE email = '" + email + "'")`. JPA does not protect you when you concatenate.

---

## 4. Where ORMs Stop Protecting You

Placeholders bind **values**. They cannot bind **identifiers** (table names, column names) or **structural keywords** (`ASC`/`DESC`, `LIMIT N`, `IN`-list arity). When the application needs to vary those, the ORM offers no built-in defense.

### 4.1 ORDER BY column injection

```typescript
// Vulnerable
const sortBy = req.query.sort;          // attacker-controlled
const sortDir = req.query.dir;
const rows = await pool.query(
  `SELECT id, email FROM users ORDER BY ${sortBy} ${sortDir} LIMIT 50`
);
```

Placeholders cannot fix this because the SQL parser must see the column name *before* planning. Defense is an **allowlist**:

```typescript
const SORT_COLUMNS = new Set(['created_at', 'email', 'name']);
const SORT_DIRS = new Set(['ASC', 'DESC']);

const sortBy = SORT_COLUMNS.has(req.query.sort as string) ? req.query.sort : 'created_at';
const sortDir = SORT_DIRS.has(req.query.dir as string) ? req.query.dir : 'ASC';

const rows = await pool.query(
  `SELECT id, email FROM users ORDER BY ${sortBy} ${sortDir} LIMIT 50`
);
```

The string interpolation is now safe because the value can only be one of a handful of known constants.

### 4.2 Dynamic table or column names

Same problem, same defense — never concatenate a user-controlled identifier; map an opaque key (`"recent"`, `"top"`) to a known table name on the server side.

### 4.3 IN-list arity

`WHERE id IN (?, ?, ?)` requires you to know the arity at prepare time. Build the placeholder list from the array length, not from the array contents:

```typescript
// node-postgres: build placeholders, bind values separately
const ids = req.body.ids;                    // array of UUIDs
const placeholders = ids.map((_, i) => `$${i + 1}`).join(', ');
const rows = await pool.query(
  `SELECT id, email FROM users WHERE id IN (${placeholders})`,
  ids
);
```

PostgreSQL also supports the `= ANY($1::uuid[])` form, which avoids dynamic placeholder construction entirely:

```typescript
await pool.query(
  'SELECT id, email FROM users WHERE id = ANY($1::uuid[])',
  [ids]
);
```

JDBC: PostgreSQL JDBC supports `setArray`, MySQL Connector/J does not — so MySQL apps usually build the placeholder list as in the first example above. Either is fine; concatenating values into the string is not.

### 4.4 LIKE and full-text patterns

`LIKE` patterns are still values, so they bind safely — but the user can still inject `%` or `_` wildcards. If the application semantically wants prefix search, escape `%`, `_`, and `\` in the application before binding:

```typescript
const escaped = raw.replace(/[\\%_]/g, ch => '\\' + ch);
await pool.query(
  "SELECT id FROM users WHERE name LIKE $1 || '%' ESCAPE '\\'",
  [escaped]
);
```

This is *not* an injection bug — it's a wildcard-confusion bug — but it travels in the same code path and is worth fixing in the same review.

---

## 5. Second-Order SQL Injection

A second-order bug stores attacker-controlled input safely on the first request, then later concatenates the stored value into a query on a second request. The first query was parameterized; the second wasn't. Reviewers miss it because the offending request looks innocent.

```typescript
// Step 1: registration — safe insert
await pool.query(
  'INSERT INTO users (name, email) VALUES ($1, $2)',
  [req.body.name, req.body.email]   // name = "admin' --"
);

// Step 2: later, in an audit job — unsafe usage of the stored name
const me = await currentUser();
const audit = await pool.query(
  `SELECT * FROM audit_log WHERE actor = '${me.name}'`
);
```

The fix is the same as §2 — the second query must also be parameterized. The lesson is that "trusted" data is the data you wrote yourself today; everything from a database column is still untrusted in the parse step. Treat every column-derived string as if it just came over the wire.

---

## 6. Blind SQL Injection — Boolean and Time-Based

When the response doesn't echo query results or errors, attackers exfiltrate one bit at a time. Two common shapes:

**Boolean-based.** The injected predicate makes the response either succeed or fail (200 vs 500, or a different content length). The attacker iterates: "is the first character of the password hash > 'a'?" — repeat until extracted.

**Time-based.** The injected predicate calls `pg_sleep(5)` (Postgres), `SLEEP(5)` (MySQL), or `WAITFOR DELAY '0:0:5'` (MSSQL) when true. The attacker measures response latency.

Both succeed against an application that has *only one query path that touches user input*, so generic defense is the same as elsewhere — parameterize, and don't let an attacker influence a predicate's structure. Per-row latency budgets and the WAF/pattern detection covered in §9 catch the noisy variants in production, but the only deterministic fix is to remove the injection point.

---

## 7. NoSQL Injection

NoSQL stores have their own injection class — usually operator injection rather than syntax injection. The two most common vectors:

### 7.1 Mongoose / MongoDB `$where`

`$where` evaluates a JavaScript predicate on each document. If user input lands inside the predicate, it executes server-side JavaScript:

```javascript
// Vulnerable: req.body.name is concatenated into a JS expression
await User.find({ $where: `this.name === '${req.body.name}'` });
```

An attacker submits `'; while(true){}; var x = '` and creates an unbounded loop on the server. Or `' || '1' === '1`, returning every document. The vulnerability class has shipped in production libraries recently — Mongoose patched a top-level `$where` injection in 8.8.3 (CVE-2024-53900), and an incomplete fix was followed by a nested-`$where` bypass patched in 8.9.5 (CVE-2025-23061). Both are real, both shipped, both are recent.

Mitigations:
- Disable server-side JavaScript at the database — set `security.javascriptEnabled: false` in `mongod.conf` (this hardens the server even if the application slips up).
- Replace `$where` JS predicates with structured operators (`$expr`, `$eq`, `$gt`).
- In Mongoose, treat `$where` (and `populate({ match: { $where: ... } })`) as if it were `eval`.

### 7.2 Operator injection from JSON bodies

If the framework parses JSON and you spread it directly into a query, a client can submit operator objects:

```javascript
// Express + Mongo native driver
app.post('/login', async (req, res) => {
  const user = await db.collection('users').findOne({
    email: req.body.email,
    password: req.body.password   // expected: a string
  });
  // ...
});
```

Sending `{"email": "a@b.c", "password": {"$ne": null}}` matches any non-null password. The same trick works in Mongoose unless the schema's `cast` rejects non-string input or the input is explicitly coerced (`String(req.body.password)`).

Defense: validate the request body shape before query construction — `zod`, `class-validator`, or Mongoose schema casting all work. Reject objects where strings were expected.

---

## 8. Stored Procedures — Help and Limits

Stored procedures *can* help when the procedure body is fully static — the application calls the procedure with bound parameters and never builds SQL itself. They do not help when the procedure body builds dynamic SQL (`EXECUTE`, `sp_executesql`, `EXEC()`) from its arguments. The Microsoft documentation for `sp_executesql` is explicit that injection is possible if parameters are not used; the same caveat applies to PL/pgSQL `EXECUTE`.

Treat a stored procedure as a black box that has the same SQLi rules as application code. If it builds dynamic SQL, it must use parameterized execution (`EXECUTE ... USING` in PL/pgSQL, parameterized `sp_executesql` in T-SQL).

---

## 9. Defense in Depth

Parameterization is necessary and sufficient for the parse-step bug. Defense in depth limits the blast radius when something else goes wrong (a new endpoint added without review, a `${}` slip in a MyBatis mapper, a stored second-order value).

### 9.1 Least-privilege database users

The application's runtime DB user should be able to do exactly the queries the application needs and no more. Practical separation:

- **Migration user** — owns DDL, runs only at deploy time.
- **App user** — `INSERT/UPDATE/DELETE/SELECT` on the application's tables, no `DROP`, no `CREATE`, no superuser.
- **Read-replica user** — `SELECT`-only on a replica, used by reporting and analytics.

A successful injection against the app user can corrupt application data; against a read-replica reporter, the worst case is exfiltration. PostgreSQL's `REVOKE ALL ON ALL TABLES IN SCHEMA public FROM PUBLIC` plus per-role grants is the canonical setup; the docs walk through it.

### 9.2 Allowlist for dynamic identifiers

Repeating §4: every place the code interpolates a column, table, or direction must check the value against a server-defined allowlist. If the allowlist is "anything the user sent," there is no allowlist.

### 9.3 Statement and query timeouts

Set `statement_timeout` (Postgres) or `MAX_EXECUTION_TIME` (MySQL) at the connection pool level. A 30-second cap on a query that should take 50 ms cuts time-based blind SQLi from "exfiltrate the database in days" to "exfiltrate the database in years," and fails noisily. This is also good operational hygiene unrelated to injection.

### 9.4 WAFs as a compensating control only

Cloud WAFs (AWS WAF, Cloudflare, etc.) ship managed SQLi rule sets that catch obvious payloads (`UNION SELECT`, `' OR 1=1`). They are useful — but they are pattern matchers, and pattern matchers are routinely bypassed by encoding tricks, comments, and case shifts. Treat the WAF as a tripwire that buys you time to fix the real bug, not as a substitute for parameterization.

### 9.5 Schema validation at the edge

Reject the request before it reaches the query builder. `zod`, `joi`, `class-validator`, or Spring's `@Valid` with Bean Validation all let you say "this field is a UUID" or "this field is one of `created_at`, `email`, `name`." A validated request narrows the attack surface without relying on downstream code being perfect.

---

## 10. Detection in Production

You will not catch every injection in code review. The following give you a fighting chance to notice one in production.

### 10.1 `pg_stat_statements` anomaly review

PostgreSQL's `pg_stat_statements` extension records every query shape (with constants normalized to `$1`, `$2`). A weekly review for **new query shapes that did not exist last week** surfaces:

- A new endpoint shipped without a parameterized rewrite.
- An attacker probing with `' OR 1=1 --` — the resulting query shape is brand new and gets logged.
- A second-order bug surfacing when the stored payload finally fires.

This works because parameterized queries normalize to a stable shape; concatenated queries do not.

### 10.2 Structured query logging

Log the query template (with placeholders) and the parameter count, never the values themselves (PII risk, log-injection risk). With trace IDs, this lets you reconstruct a request flow without exposing the underlying data. Spring's `p6spy` or `datasource-proxy`, and node-postgres's per-query event hook, both support this pattern.

### 10.3 Database audit triggers for sensitive tables

For high-value tables (credentials, payment details, admin actions), emit an audit row on every read. Anomalies — bulk reads, off-hours reads, reads from unexpected service accounts — are visible in the audit table even when the application logs are clean. This catches the *consequence* of an injection, not the injection itself.

### 10.4 Alerting on driver errors

A rising rate of `42601` (Postgres syntax error) or `1064` (MySQL syntax error) is one of the loudest possible signals that something is shoving raw input into a query. Wire these to your error tracker (Sentry, Datadog) with the query template (not the values) attached.

---

## 11. Notable Incidents and CVEs

These are real, public, and worth knowing for both interview signal and production instinct.

| Year | Target | Vector | Outcome |
|------|--------|--------|---------|
| 2008 | **Heartland Payment Systems** | SQL injection on a web login page (deployed 8 years earlier), used to install a packet sniffer on the payment processing network | ~134M payment cards exposed; one of the largest breaches of the era; principal attacker (Albert Gonzalez) prosecuted |
| 2015 | **TalkTalk** | SQL injection against legacy pages inherited from a Tiscali acquisition; underlying database software was unpatched | ~157K customer records, ~16K bank account details; £400,000 ICO fine — at the time the largest ever |
| 2017 | **Equifax** | *Not SQLi* — Apache Struts RCE (CVE-2017-5638) via an unhandled exception in the Jakarta Multipart parser | ~147M records; included here as a reminder that input validation is not just SQL — every parser at a trust boundary deserves the same scrutiny |
| 2022 | **PostgreSQL JDBC Driver** | CVE-2022-31197 — `ResultSet.refreshRow()` did not escape column names, allowing SQL injection through crafted column names | Patched in pgjdbc 42.4.1 / 42.2.26; reminder that even the driver code path can be the injection point |
| 2024 | **Mongoose (Node.js MongoDB ODM)** | CVE-2024-53900 — `$where` could be improperly used in `match`, leading to search injection | Patched in 8.8.3; followed by CVE-2025-23061 (incomplete fix; nested `$where` bypass via `$or`/`populate`), patched in 8.9.5 |

The pattern is depressingly consistent: a forgotten code path (legacy page, refresh hook, populate match) carries unparameterized input, and an attacker who reads release notes finds it before the team that owns the code does.

---

## Related

- [OWASP Top 10 (2021) — A03 Injection](../fundamentals/01-owasp-top-10.md) — the broader taxonomy this doc deepens
- [Threat Modeling with STRIDE](../fundamentals/02-threat-modeling-stride.md) — "Tampering" and "Information Disclosure" as the STRIDE buckets SQLi lives in
- [Authentication vs Authorization](../fundamentals/03-authentication-vs-authorization.md) — least-privilege DB users (§9.1) connect to the same identity model
- [JWT Design and Pitfalls](../fundamentals/05-jwt-design-and-pitfalls.md) — sibling Tier 1 doc on a different injection-adjacent class
- [Database Indexing and Query Plans](../../database/query-optimization/) — `pg_stat_statements` (§10.1) used here as security telemetry, used there for performance
- [JPA Transactions and Entity Lifecycle](../../java/jpa-transactions.md) — JPA-side context for the Hibernate query examples in §3.3
- [Spring Security Bug Spotting](../../java/spring-security-bug-spotting.md) — adjacent practice doc on Spring-specific security bugs

## References

- [OWASP — A03:2021 Injection](https://owasp.org/Top10/A03_2021-Injection/) — primary OWASP entry for this class
- [OWASP Cheat Sheet — SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html) — the canonical defensive checklist
- [OWASP Web Security Testing Guide — NoSQL Injection](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/05.6-Testing_for_NoSQL_Injection) — testing methodology used in §7
- [PostgreSQL — Frontend/Backend Protocol](https://www.postgresql.org/docs/current/protocol.html) — `Parse` / `Bind` messages described in §2.1
- [PostgreSQL — `pg_stat_statements`](https://www.postgresql.org/docs/current/pgstatstatements.html) — query shape normalization used in §10.1
- [PostgreSQL JDBC Driver — Security advisory CVE-2022-31197](https://nvd.nist.gov/vuln/detail/CVE-2022-31197) — `refreshRow()` column-name injection
- [Prisma — Raw Database Access](https://www.prisma.io/docs/orm/prisma-client/queries/raw-database-access) — `$queryRaw` vs `$queryRawUnsafe` distinction in §3.1
- [MyBatis — XML Configuration (`#{}` vs `${}`)](https://mybatis.org/mybatis-3/sqlmap-xml.html) — placeholder syntax behind §3.2
- [MongoDB — `$where` operator](https://www.mongodb.com/docs/manual/reference/operator/query/where/) — server-side JavaScript evaluation discussed in §7.1
- [Mongoose — CVE-2024-53900 advisory](https://nvd.nist.gov/vuln/detail/CVE-2024-53900) — `$where`-in-`match` injection
- [Mongoose — CVE-2025-23061 advisory](https://nvd.nist.gov/vuln/detail/CVE-2025-23061) — nested `$where` bypass in `populate`
- [Apache Struts — CVE-2017-5638](https://nvd.nist.gov/vuln/detail/CVE-2017-5638) — Equifax 2017 root cause; cited in §11 to distinguish from SQLi
- [ICO — TalkTalk cyber attack: how the investigation unfolded](https://ico.org.uk/about-the-ico/media-centre/talktalk-cyber-attack-how-the-ico-investigation-unfolded/) — UK regulator's account of the 2015 breach and £400K fine
- [Computerworld — SQL injection attacks led to Heartland, Hannaford breaches](https://www.computerworld.com/article/1555445/sql-injection-attacks-led-to-heartland-hannaford-breaches.html) — contemporaneous reporting on the 2008 Heartland vector
- [PortSwigger Web Security Academy — Blind SQL Injection](https://portswigger.net/web-security/sql-injection/blind) — material behind §6
