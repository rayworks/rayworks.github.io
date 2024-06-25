---
layout: post
title: DB Migration: Ensuring Smooth Transitions
tags:
  - code
  - Database
  - Android
  - Python
  - syntax highlighting
---

## Introduction

As a Release Owner, it is crucial to address any potential database (DB) migration issues early on rather than fixing them at the last moment. To achieve this, we need to handle DB changes effectively and ensure proper testing. Here are some approaches to consider:

## Raw SQLite

If your project has a long history and still relies on raw SQLite, you can utilize the `onUpgrade()` method within `SQLiteOpenHelper`. This method is triggered when a newer DB version is detected during an upgrade. However, testing all the changes within this method can be inconvenient.

To abstract and manage these changes more efficiently, you can employ a solution like the one described in Rebecca's blog post on [correctly using onUpgrade() in Android SQLite Database](https://web.archive.org/web/20160103195237/http://riggaroo.co.za/android-sqlite-database-use-onupgrade-correctly/). This approach involves creating separate SQL script files for each DB version transition.

Here's how it works:

> 1. Create a SQL file named `from_2_to_3.sql`, corresponding to the version you are upgrading from and to.
> 2. Write the necessary SQL statements in the file to perform the required changes.
> 3. Increase the `DATABASE_VERSION` in your code to the new version number.
> 4. Ensure you update the create script for fresh installations of your app.
> 5. During an upgrade, the migration process will execute the scripts stored in the `res/raw/` folder to upgrade the DB to the latest version.

> This method offers the benefit of avoiding upgrade scripts that rely on variables defined in code. It also provides a clear overview of the changes made per version of your database.

To test this approach, you can keep historical DB files in the test assets folder. By copying the DB file with a lower version to replace the app's database file, you can create a new `DatabaseOpenHelper`. Invoking its `getWritableDatabase()` method will open the database, triggering the `onUpgrade` method, which will read and execute the migration scripts. If any issues occur with the migration logic, a `SQLException` will be thrown.

While this approach is effective, it has a drawback from a testing perspective, as you need to maintain historical DB files for testing purposes.

## ORMs (Object-Relational Mapping)

Let's explore how ORMs, such as [Room](https://developer.android.com/training/data-storage/room), handle DB migration.

Room simplifies the migration process by introducing an abstract class called `Migration`. This class includes the `startVersion`, `endVersion`, and an abstract `migrate()` method. Room also provides a specific class called `MigrationTestHelper`, which facilitates testing by mocking the behavior of `SQLiteDatabase` and providing an independent interface called `SupportSQLiteDatabase`.

For instance, using `MigrationTestHelper`, you can run migrations and validate the DB schema changes in a test scenario:

```java
migrationTestHelper.runMigrationsAndValidate(TEST_DB_NAME, 3, true, MIGRATION_1_2, MIGRATION_2_3);
```

In this test, you can create a lower version of the DB, set up the required `Migrations`, and open the database with the higher version. This process automatically validates the DB schema changes, allowing you to also verify the data's integrity.

Other DB frameworks, such as `DBFlow` and `SqlDelight`, likely have their own strategies for handling DB migration. However, the underlying concept remains promising.

## Flask-Migrate (Python)
In the Python world, the Flask web framework offers a solution called [Flask-Migrate](https://github.com/miguelgrinberg/Flask-Migrate), which helps track changes to the DB schema and apply incremental changes.

Flask-Migrate provides two related subcommands. Using the `flask db migrate` command generates an automatic migration script. You can then apply these changes to the database using the `flask db upgrade` command. This approach proves beneficial when comparing schemas and reviewing the generated scripts.

By leveraging these strategies, you can ensure smooth transitions during DB migration and minimize potential issues in your projects.

## Ref

* [AndroidDatabaseUpgrades](https://github.com/riggaroo/AndroidDatabaseUpgrades)
  
* [Room : migrate your database](https://developer.android.com/training/data-storage/room/migrating-db-versions#test)

* [Room migration sample](https://github.com/android/architecture-components-samples/tree/main/PersistenceMigrationsSample)