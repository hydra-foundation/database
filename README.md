# Hydra Database

A thin data-access seam over PDO and a raw-SQL migration runner. No ORM, no
query builder, no fluent anything — repositories write their own SQL and pass
bound parameters. This package ships the *mechanism*; the app owns its
repositories, entities, and migration files.

## The connection seam

`Contracts\ConnectionInterface` is a prepared-statement-only surface:
`select`, `selectOne`, `execute`, `lastInsertId`. Repositories depend on the
interface, never on PDO, so the engine is swappable — the same move
`Hydra\View\Contracts\ViewInterface` makes for templating.

```php
use Hydra\Database\PdoConnection;

$db = new PdoConnection($pdo); // $pdo built by the app's service provider
$rows = $db->select('SELECT * FROM posts WHERE published = ?', [1]);
$one  = $db->selectOne('SELECT * FROM users WHERE email = ?', [$email]);
$n    = $db->execute('UPDATE users SET name = ? WHERE id = ?', [$name, $id]);
```

`PdoConnection` prepares every statement, so all values reach the driver as
bound parameters — there is no string-built SQL path. The PDO handle itself is
constructed by the app (with its chosen error mode, fetch mode, prepare
settings), which keeps this class driver-agnostic and trivially testable against
in-memory sqlite.

`lastInsertId()` returns a **string** — PDO's native surface — so UUID/string
primary keys and 64-bit ids beyond `PHP_INT_MAX` on 32-bit builds pass through
uncorrupted. If your schema uses integer PKs, cast at the call site, where that
schema assumption actually lives:

```php
$db->execute('INSERT INTO users (username) VALUES (?)', [$username]);
$id = (int) $db->lastInsertId(); // this table's PK is an integer — your call
```

## Transactions

`transaction(callable): mixed` runs the callable inside a transaction and
returns its value. A clean return commits; any throwable rolls back and
propagates unchanged, so callers never see a half-applied write:

```php
$id = $db->transaction(function (ConnectionInterface $db) use ($input) {
    $db->execute('INSERT INTO orders (user_id) VALUES (?)', [$input->int('user_id')]);
    $orderId = $db->lastInsertId();
    $db->execute('INSERT INTO order_events (order_id, event) VALUES (?, ?)', [$orderId, 'created']);

    return $orderId;
});
```

Calls are re-entrant: a `transaction()` nested inside another joins the outer
transaction rather than opening a second (PDO has no true nesting; this seam
ships no savepoints). The outermost call owns begin/commit/rollback, and a
throw anywhere inside rolls the whole thing back — so repositories that each
wrap their own writes compose atomically.

One honest sharp edge: on MySQL/MariaDB, DDL statements (`CREATE`/`ALTER`/
`DROP`) trigger an implicit commit, so DDL inside `transaction()` is not rolled
back — the same caveat the migration runner documents.

## Migrations

`MigrationRunner` applies plain `.sql` files in lexical order and records each in
a `migrations` table, so re-runs are no-ops. Forward-only by design — there are
no down migrations (rollback in production is mostly fiction; in dev you
`fresh()` from scratch).

```php
$runner = new MigrationRunner($pdo, __DIR__ . '/database/migrations', $driver);
$runner->run();     // apply pending, return applied filenames
$runner->status();  // [{filename, applied}, …]
$runner->fresh();   // drop everything, re-apply (destructive — guard it)
```

It takes the raw PDO rather than the connection seam on purpose: DDL is
multi-statement and unparameterised, so it runs through `PDO::exec`, outside the
prepared-only surface repositories use. The `$driver` argument switches the
drop-all dialect (MariaDB `information_schema` vs sqlite `sqlite_master`), so
`fresh()` stays exercisable under the sqlite test driver while production targets
MariaDB.

> **Caveat:** MariaDB has no transactional DDL — a migration that fails halfway
> leaves partial state. Keep each migration to one logical change.

## What's app-owned

The PDO construction, the migration `.sql` files, repositories and entities all
live in the app. This package is only the connection seam and the runner — the
two pieces that are identical across every app and were previously copied.
