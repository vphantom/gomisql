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
- `-p <path>` Location of the SQL files [default: `./migrations/`]
- `-b <cmd>` Name of back-end command [default: `mysql`]
- `-a <args>` Arguments to pass to back-end command [default: `--batch`]

Several environment variables can also be used, although command line options have priority:

- `GOMISQL_DEBUG` (set non-empty to enable)
- `GOMISQL_PATH`
- `GOMISQL_BACKEND`
- `GOMISQL_ARGS`

#### SQLite3 Example

```sh
gomisql -b sqlite3 -a testdb.sqlite list
```

#### MySQL Example

```sh
# Assuming you are configured for non-interactive authentication
gomisql -a "--user someuser somedb" list
```

#### PostgreSQL Example

```sh
gomisql -b psql -a "-d somedb -h somehost -U someuser" list
```

### Commands

#### `gomisql [opts] list`

Run all validations to discover which are and aren't deployed.

#### `gomisql [opts] create <name>`

Create empty migration `name` from handy template.

#### `gomisql [opts] deploy [<name>]`

Deploy migration `name` and dependencies, or all avaliable if not specified.

#### `gomisql [opts] revert <name>`

Revert migration `name`.  Aborts if any deployed migration depends on `name`, displaying which.

## Migration Files

A migration file is a standard SQL file with special comments.  Standard multiline `/* ... */` comments are ignored.  Single-line `-- ` comments are considered to be the start of a new section, so they cannot be used inside of a section.

### `-- #Dependencies: ...`

Optional.  Explicitly name other migration file basenames (without the `.sql` extension) which must be deployed and valid before attempting to deploy this one.

### `-- #Deploy:`

Mandatory.  Begins SQL code to run when deploying the migration.  Ends at the next single-line comment.

### `-- #Verify:`

Mandatory.  Begins SQL code to run to prove that a migration was successfully deployed.  Ends at the next single-line comment.  Verifications are run after deployments and whenever dependencies need checking.  **CAUTION:** While you may technically omit this section, doing so means the migration is considered to be already deployed and is thus not deployable!

To test whether a table or some columns exist, just select them with a false condition, as the SQL back-end will fail and let us know:

```sql
BEGIN;
/* Prove that foo.col1, foo.col2 and foo.col3 exist in schema. */
SELECT col1, col2, col3 FROM foo WHERE 0;
ROLLBACK;
```

To test whether a specific row exists, make the back-end output `#FAIL` which will be recognized as a failure.  Forcing the presence of results with `COUNT(*)` does the trick, along with `COALESCE()`.  Be sure to use one distinct `SELECT` per row to check, so that any missing row causes a failure.

```sql
BEGIN;
/* Output '#FAIL' if user 23 does not exist. */
SELECT COALESCE(id, '#FAIL', COUNT(*)) AS id FROM users WHERE id='23';
ROLLBACK;
```

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

BEGIN;
CREATE TABLE IF NOT EXISTS users (
    `id` INTEGER UNSIGNED NOT NULL AUTO_INCREMENT,
    `name` VARCHAR(255) NOT NULL DEFAULT '',
    PRIMARY KEY (id)
);
COMMIT;

-- #Verify:

BEGIN;
SELECT id, name FROM users WHERE 0;
ROLLBACK;

-- #Revert:

BEGIN;
DROP TABLE users;
COMMIT;

-- This is ignored

This is ignored
```

Running the following would cause migrations foo and bar, then baz to be installed if missing:

```sh
gomisql -y -a "--batch --user dbuser" deploy baz
```

## FUTURE ENHANCEMENTS

Dependencies are installed outright during deployment; it would be better if a list was displayed and the user was asked to accept, along with a `-y` option to assume "yes" (like `apt-get`).

Similarly, instead of aborting when a reversion is impossible, it would be better to display the list of migrations which would need to be reverted and prompt for approval.

Both the above imply maintaining a list of migrations to be deployed or removed in memory instead of doing it as we go, which while feasible isn't an absolute necessity for now.


## ACKNOWLEDGEMENTS

Graph X Design Inc. https://www.gxd.ca/ sponsored this project.


## LICENSE AND COPYRIGHT

Copyright (c) 2018 Stéphane Lavergne <https://github.com/vphantom>

This program is distributed under the MIT (X11) License:
http://www.opensource.org/licenses/mit-license.php

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
