---
title: A Sqitch "Declare Bankruptcy" Prototype
author: Dave Rolsky
type: post
date: 2020-10-15T18:09:00
url: /2020/10/15/sqitch-declare-bankruptcy-prototype
---
We've all been there. You've been using [Sqitch](https://sqitch.org/) for a couple years. You have a few hundred migrations. You've taken advantage of Sqitch's [cool dependency features](https://sqitch.org/docs/manual/sqitch-add/) to make sure that your migrations can always be run from scratch. You don't have a damn clue what your database schema looks like any more.

It all started off great. Your first migration set up your initial schema. It was a bunch of `CREATE TABLE this` and `CREATE FUNCTION that` statements. You added a table or two in the next few migrations, maybe dropped a column. You could read through those first couple files and easily understand the schema. But years later, you've applied more `ALTER TABLE` statements than you can count. The only definitive schema is what's running in the database after you apply all your migrations, but it's not easy to look at a table and see all its indexes, constraints, and triggers from `psql`[^1].

<small>(As an aside, none of this is a criticism of Sqitch itself. This is just the nature of a long-running project with lots of migrations. Sqitch is working exactly as intended here.)</small>

What if you could start over? A clean slate? All of your sins forgiven, the constraints you added only to drop in the next migration wiped away. All of your stored procedures laid out in one easy to browse place. What if you could ... declare Sqitch bankruptcy?

Well, friends, I'm here to tell you that you can... if your repo layout looks exactly what we use at ActiveState. See [sqitch-reset](...)!

One fun thing we do at [ActiveState](https://www.activestate.com/) (hey, [we're hiring](https://www.activestate.com/company/careers/)!) is called Innovation Days. Once each quarter, the whole company gets two days to do whatever they feel like. Some people read some books or take classes. Some folks hack on personal projects. Some folks refactor something in our codebase or try adding a pet feature. Sor for this quarter's Innovation Days I took a stab at this Sqitch problem.

I stole this idea from Curtis "Ovid" Poe, who had [discussed this on the sqitch-hackers list](https://groups.google.com/g/sqitch-hackers/c/vUzQcn0F9ig/m/ZCwdVs-6CwAJ) last year.

The script I wrote does a few things:

1. Runs all of your migrations against an empty database from scratch.
2. Runs `pg_dump` to dump the schema and data from that database.
3. Moves your existing `sqitch.plan` and migration files to an `archive/YYYY-MM-dd` directory.
4. Starts a "new" Sqitch project in the same directory where the old project was.
5. Moves the dump file to a deployment file at `deploy/reset-YYYY-MM-dd.sql`.
6. Extracts all the stored procedures from a dump to a separate directory, with each procedure in its own file (I'll explain why later) and updates the dump to load them using the `psql` `\ir ...` command.
7. Creates mostly empty verify and revert files just to have something there.
8. Runs `sqitch add` to add the new schema dump as your first migration.
9. Spews out some SQL and a Sqitch command to finish the reset. This has to be applied by hand, since it involves changing the `sqitch` schema, which you can't really do as part of a Sqitch migration. It has been to done _before_ the first migration is run. The `sqitch` command it gives you will log the `reset-YYYY-MM-dd` migration as having been applied without actually applying it. You'll need to run this SQL and this command against your production DB.

**Run this at your own risk! Test thoroughly in a staging environment before you do this. Take a backup first. Heck, take two! Seriously, I haven't even tried this on our production system as of this moment. This is very much pre-alpha code!**

So what's the deal with putting stored procedures in their own files? We end up with a directory structure like this after running the tool:

```sh
db
├── archive
│   └── 2020-10-15
│       └── deploy
│           ├── old-migration-1.sql
│           └── old-migration-2.sql
├── deploy
│   └── reset-2020-10-15.sql
├── init.sql
├── lib
│   └── for-production
│       ├── some-func.v1.sql
│       └── another-func.v1.sql
├── revert
│   └── reset-2020-10-15.sql
├── sqitch.conf
├── sqitch.plan
└── verify
    └── reset-2020-10-15.sql
```

If you have a bunch of stored procedures in your migrations, it can be _really_ hard to understand how a migration changes them. You just have a bunch of inline `CREATE OR REPLACE FUNCTION ...` statements scattered through your SQL. You can't diff the functions easily, and if you grep for a function, it's really hard to know which version is the latest.

But if you put these in their own dir and version them by hand, that problem is solved. You can run plain old `diff -u` on two versions of the file to see how it changed, and it's easy to know what the latest version is.

This isn't as nice as _real_ version control, but you don't edit code in place with database migrations.

If I have time in the future, I'd like to turn this prototype into something that can go in the sqitch core. There's a lot more work involved in that, especially since Sqitch supports so many different databases.

[^1]: I'm going to assume that you're using Postgres because you're not a monster, are you?
