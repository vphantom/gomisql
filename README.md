# SQL Good Migrations (gomisql)

[![license](https://img.shields.io/github/license/vphantom/gomisql.svg?style=plastic)]() [![GitHub release](https://img.shields.io/github/release/vphantom/gomisql.svg?style=plastic)]()

Tiny stand-alone database upgrade/downgrade manager with dependency support.

- Single file, runs on any POSIX shell (sh, ash, dash, ksh, bash...)
- Compatible with stdin-friendly backends (MySQL/MariaDB, SQLite3, PostgreSQL...)
- Each migration is a single SQL file
- No need for numeric versioning: each migration is named arbitrarily
- Migrations can depend on other migrations
- Migrations are verified for success
- Migrations can be reverted

## Why Another SQL Migration Manager?

I wanted a migration tool similar in functionality to the excellent https://sqitch.org/ but without the run-time dependencies on Perl and a lot of modules.  I looked into https://github.com/mbucc/shmig which is a shell script, but doesn't support dependencies between migrations themselves.  I ended up "rolling my own" and gomiSQL was born.

## Installation & Usage

Just download the stand-alone `gomisql` shell script and make it executable. ;-)

### Options

- `-h` Display help
- `-d` Display all executed SQL and back-end output
- `-y` Automatic "yes" to prompts
- `-p <path>` Location of the SQL files [default: `./migrations/`]
- `-b <cmd>` Name of back-end command [default: `mysql`]
- `-a <args>` Arguments to pass to back-end command [default: `--batch`]

#### SQLite3 Example

```sh
gomisql -b sqlite3 -a "-batch testdb.sqlite" list
```

#### MySQL Example

```sh
# Assuming you are configured for non-interactive authentication
gomisql -b mysql -a "--batch --user someuser somedb" list
```

#### PostgreSQL Example

```sh
gomisql -b psql -a "-d somedb -h somehost -w -U someuser" list
```

### Commands

#### `gomisql [opts] list`

Run all validations to discover which still need deployment.

#### `gomisql [opts] deploy [<name>]`

Deploy migration `name`, or all avaliable if not specified.  If any dependency is missing, you will be prompted with the list to be installed.  (Use `-y` to agree by default.)

#### `gomisql [opts] revert <name>`

Revert migration `name`.  If any migration depends on `name`, you will be prompted with the list to be removed.  Note that if any migration in the dependency chain is missing a `Revert` block, the migration will not be reverted.

## Migration Files

A migration file is a standard SQL file with special comments.  Standard multiline `/* ... */` comments are ignored.  Single-line `-- ` comments are considered to be the start of a new section, so they cannot be used inside of a section.

### `-- #Dependencies: ...`

Optional.  Explicitly name other migration file basenames (without the `.sql` extension) which must be deployed and valid before attempting to deploy this one.

### `-- #Deploy:`

Mandatory.  Begins SQL code to run when deploying the migration.  Ends at the next single-line comment.

### `-- #Verify:`

Mandatory.  Begins SQL code to run to prove that a migration was successfully deployed.  Ends at the next single-line comment.  Verifications are run after deployments and whenever gomisql needs to check dependencies.

### `-- #Revert:`

Optional.  Begins SQL code to run to revert the migration.  Ends at the next single-line comment.

### Example

Given the following files:

```text
./migrations/foo.sql
./migrations/bar.sql
./migrations/baz.sql
```

...and `./migrations/baz.sql` containing the following:

```sql
-- #Dependencies: foo bar

This is ignored!

-- #Deploy:

CREATE TABLE IF NOT EXISTS users (
    `id` INTEGER UNSIGNED NOT NULL AUTO_INCREMENT,
    `name` VARCHAR(255) NOT NULL DEFAULT '',
    PRIMARY KEY (id)
);

-- #Verify:

SELECT id, name FROM users WHERE 0;

-- #Revert:

DROP TABLE users;

-- This is ignored

This is ignored
```

Running the following would cause migrations foo and bar, then baz to be installed if missing:

```sh
gomisql -y -a "--batch --user dbuser" deploy baz
```

## MIT License

Copyright (c) 2018 Stéphane Lavergne <https://github.com/vphantom>

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
