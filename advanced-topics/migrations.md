# Migrations

## How migrations work

Once you get into production you'll need to synchronize model changes into the database. Typically it is unsafe to use `synchronize: true` for schema synchronization on production once you get data in your database. Here is where migrations come to help.

A migration is just a single file with sql queries to update a database schema and apply new changes to an existing database.

Let's say you already have a database and a post entity:

```typescript
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class Post {

    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    title: string;

    @Column()
    text: string;

}
```

And your entity worked in production for months without any changes. You have thousands of posts in your database.

Now you need to make a new release and rename `title` to `name`. What would you do?

You need to create a new migration with the following sql query \(postgres dialect\):

```sql
ALTER TABLE "post" ALTER COLUMN "title" RENAME TO "name";
```

Once you run this sql query your database schema is ready to work with your new codebase. TypeORM provides a place where you can write such sql queries and run them when needed. This place is called "migrations".

## Creating a new migration

**Pre-requisites**: [Installing CLI](using-cli.md#installing-cli)

Before creating a new migration you need to setup your connection options properly:

```javascript
{
    "type": "mysql",
    "host": "localhost",
    "port": 3306,
    "username": "test",
    "password": "test",
    "database": "test",
    "entities": ["entity/*.js"],
    "migrationsTableName": "custom_migration_table",
    "migrations": ["migration/*.js"],
    "cli": {
        "migrationsDir": "migration"
    }
}
```

Here we setup three options:

* `"migrationsTableName": "migrations"` - Specify this option only if you need migration table name to be different from `"migrations"`.
* `"migrations": ["migration/*.js"]` - indicates that typeorm must load migrations from the given "migration" directory.
* `"cli": { "migrationsDir": "migration" }` - indicates that the CLI must create new migrations in the "migration" directory.

Once you setup connection options you can create a new migration using CLI:

```text
typeorm migration:create -n PostRefactoring
```

Here, `PostRefactoring` is the name of the migration - you can specify any name you want. After you run the command you can see a new file generated in the "migration" directory named `{TIMESTAMP}-PostRefactoring.ts` where `{TIMESTAMP}` is the current timestamp when the migration was generated. Now you can open the file and add your migration sql queries there.

You should see the following content inside your migration:

```typescript
import {MigrationInterface, QueryRunner} from "typeorm";

export class PostRefactoringTIMESTAMP implements MigrationInterface {

    async up(queryRunner: QueryRunner): Promise<void> {

    }

    async down(queryRunner: QueryRunner): Promise<void> {

    }


}
```

There are two methods you must fill with your migration code: `up` and `down`. `up` has to contain the code you need to perform the migration. `down` has to revert whatever `up` changed. `down` method is used to revert the last migration.

Inside both `up` and `down` you have a `QueryRunner` object. All database operations are executed using this object. Learn more about [query runner](query-runner.md).

Let's see what the migration looks like with our `Post` changes:

```typescript
import {MigrationInterface, QueryRunner} from "typeorm";

export class PostRefactoringTIMESTAMP implements MigrationInterface {

    async up(queryRunner: QueryRunner): Promise<void> {
        await queryRunner.query(`ALTER TABLE "post" RENAME COLUMN "title" TO "name"`);
    }

    async down(queryRunner: QueryRunner): Promise<void> {
        await queryRunner.query(`ALTER TABLE "post" RENAME COLUMN "name" TO "title"`); // reverts things made in "up" method
    }
}
```

## Running and reverting migrations

Once you have a migration to run on production, you can run them using a CLI command:

```text
typeorm migration:run
```

**`typeorm migration:create` and `typeorm migration:generate` will create `.ts` files. The `migration:run` and `migration:revert` commands only work on `.js` files. Thus the typescript files need to be compiled before running the commands.** Alternatively you can use `ts-node` in conjunction with `typeorm` to run `.ts` migration files.

Example with `ts-node`:

```text
ts-node ./node_modules/typeorm/cli.js migration:run
```

This command will execute all pending migrations and run them in a sequence ordered by their timestamps. This means all sql queries written in the `up` methods of your created migrations will be executed. That's all! Now you have your database schema up-to-date.

If for some reason you want to revert the changes, you can run:

```text
typeorm migration:revert
```

This command will execute `down` in the latest executed migration. If you need to revert multiple migrations you must call this command multiple times.

## Generating migrations

TypeORM is able to automatically generate migration files with schema changes you made.

Let's say you have a `Post` entity with a `title` column, and you have changed the name `title` to `name`. You can run following command:

```text
typeorm migration:generate -n PostRefactoring
```

And it will generate a new migration called `{TIMESTAMP}-PostRefactoring.ts` with the following content:

```typescript
import {MigrationInterface, QueryRunner} from "typeorm";

export class PostRefactoringTIMESTAMP implements MigrationInterface {

    async up(queryRunner: QueryRunner): Promise<void> {
        await queryRunner.query(`ALTER TABLE "post" ALTER COLUMN "title" RENAME TO "name"`);
    }

    async down(queryRunner: QueryRunner): Promise<void> {
        await queryRunner.query(`ALTER TABLE "post" ALTER COLUMN "name" RENAME TO "title"`);
    }


}
```

See, you don't need to write the queries on your own. The rule of thumb for generating migrations is that you generate them after "each" change you made to your models. To apply multi-line formatting to your generated migration queries, use the `p` \(alias for `--pretty`\) flag.

## Connection option

If you need to run/revert your migrations for another connection rather than the default, use the `-c` \(alias for `--connection`\) and pass the config name as an argument

```text
typeorm -c <your-config-name> migration:{run|revert}
```

## Using migration API to write migrations

In order to use an API to change a database schema you can use `QueryRunner`.

Example:

```typescript
import {MigrationInterface, QueryRunner, Table, TableIndex, TableColumn, TableForeignKey } from "typeorm";

export class QuestionRefactoringTIMESTAMP implements MigrationInterface {

    async up(queryRunner: QueryRunner): Promise<void> {
        await queryRunner.createTable(new Table({
            name: "question",
            columns: [
                {
                    name: "id",
                    type: "int",
                    isPrimary: true
                },
                {
                    name: "name",
                    type: "varchar",
                }
            ]
        }), true)

        await queryRunner.createIndex("question", new TableIndex({
            name: "IDX_QUESTION_NAME",
            columnNames: ["name"]
        }));

        await queryRunner.createTable(new Table({
            name: "answer",
            columns: [
                {
                    name: "id",
                    type: "int",
                    isPrimary: true
                },
                {
                    name: "name",
                    type: "varchar",
                },
                {
                  name: 'created_at',
                  type: 'timestamp',
                  default: 'now()'
                }
            ]
        }), true);

        await queryRunner.addColumn("answer", new TableColumn({
            name: "questionId",
            type: "int"
        }));

        await queryRunner.createForeignKey("answer", new TableForeignKey({
            columnNames: ["questionId"],
            referencedColumnNames: ["id"],
            referencedTableName: "question",
            onDelete: "CASCADE"
        }));
    }

    async down(queryRunner: QueryRunner): Promise<void> {
        const table = await queryRunner.getTable("question");
        const foreignKey = table.foreignKeys.find(fk => fk.columnNames.indexOf("questionId") !== -1);
        await queryRunner.dropForeignKey("question", foreignKey);
        await queryRunner.dropColumn("question", "questionId");
        await queryRunner.dropTable("answer");
        await queryRunner.dropIndex("question", "IDX_QUESTION_NAME");
        await queryRunner.dropTable("question");
    }

}
```

```typescript
getDatabases(): Promise<string[]>
```

Returns all available database names including system databases.

```typescript
getSchemas(database?: string): Promise<string[]>
```

* `database` - If database parameter specified, returns schemas of that database

Returns all available schema names including system schemas. Useful for SQLServer and Postgres only.

```typescript
getTable(tableName: string): Promise<Table|undefined>
```

* `tableName` - name of a table to be loaded

Loads a table by a given name from the database.

```typescript
getTables(tableNames: string[]): Promise<Table[]>
```

* `tableNames` - name of a tables to be loaded

Loads a tables by a given names from the database.

```typescript
hasDatabase(database: string): Promise<boolean>
```

* `database` - name of a database to be checked

Checks if database with the given name exist.

```typescript
hasSchema(schema: string): Promise<boolean>
```

* `schema` - name of a schema to be checked

Checks if schema with the given name exist. Used only for SqlServer and Postgres.

```typescript
hasTable(table: Table|string): Promise<boolean>
```

* `table` - Table object or name

Checks if table exist.

```typescript
hasColumn(table: Table|string, columnName: string): Promise<boolean>
```

* `table` - Table object or name
* `columnName` - name of a column to be checked

Checks if column exist in the table.

```typescript
createDatabase(database: string, ifNotExist?: boolean): Promise<void>
```

* `database` - database name
* `ifNotExist` - skips creation if `true`, otherwise throws error if database already exist

Creates a new database.

```typescript
dropDatabase(database: string, ifExist?: boolean): Promise<void>
```

* `database` - database name
* `ifExist` - skips deletion if `true`, otherwise throws error if database was not found

Drops database.

```typescript
createSchema(schemaPath: string, ifNotExist?: boolean): Promise<void>
```

* `schemaPath` - schema name. For SqlServer can accept schema path \(e.g. 'dbName.schemaName'\) as parameter.

  If schema path passed, it will create schema in specified database

* `ifNotExist` - skips creation if `true`, otherwise throws error if schema already exist

Creates a new table schema.

```typescript
dropSchema(schemaPath: string, ifExist?: boolean, isCascade?: boolean): Promise<void>
```

* `schemaPath` - schema name. For SqlServer can accept schema path \(e.g. 'dbName.schemaName'\) as parameter.

  If schema path passed, it will drop schema in specified database

* `ifExist` - skips deletion if `true`, otherwise throws error if schema was not found
* `isCascade` - If `true`, automatically drop objects \(tables, functions, etc.\) that are contained in the schema.

  Used only in Postgres.

Drops a table schema.

```typescript
createTable(table: Table, ifNotExist?: boolean, createForeignKeys?: boolean, createIndices?: boolean): Promise<void>
```

* `table` - Table object.
* `ifNotExist` - skips creation if `true`, otherwise throws error if table already exist. Default `false`
* `createForeignKeys` - indicates whether foreign keys will be created on table creation. Default `true`
* `createIndices` - indicates whether indices will be created on table creation. Default `true`

Creates a new table.

```typescript
dropTable(table: Table|string, ifExist?: boolean, dropForeignKeys?: boolean, dropIndices?: boolean): Promise<void>
```

* `table` - Table object or table name to be dropped
* `ifExist` - skips dropping if `true`, otherwise throws error if table does not exist
* `dropForeignKeys` - indicates whether foreign keys will be dropped on table deletion. Default `true`
* `dropIndices` - indicates whether indices will be dropped on table deletion. Default `true`

Drops a table.

```typescript
renameTable(oldTableOrName: Table|string, newTableName: string): Promise<void>
```

* `oldTableOrName` - old Table object or name to be renamed
* `newTableName` - new table name

Renames a table.

```typescript
addColumn(table: Table|string, column: TableColumn): Promise<void>
```

* `table` - Table object or name
* `column` - new column

Adds a new column.

```typescript
addColumns(table: Table|string, columns: TableColumn[]): Promise<void>
```

* `table` - Table object or name
* `columns` - new columns

Adds a new column.

```typescript
renameColumn(table: Table|string, oldColumnOrName: TableColumn|string, newColumnOrName: TableColumn|string): Promise<void>
```

* `table` - Table object or name
* `oldColumnOrName` - old column. Accepts TableColumn object or column name
* `newColumnOrName` - new column. Accepts TableColumn object or column name

Renames a column.

```typescript
changeColumn(table: Table|string, oldColumn: TableColumn|string, newColumn: TableColumn): Promise<void>
```

* `table` - Table object or name
* `oldColumn` -  old column. Accepts TableColumn object or column name
* `newColumn` -  new column. Accepts TableColumn object

Changes a column in the table.

```typescript
changeColumns(table: Table|string, changedColumns: { oldColumn: TableColumn, newColumn: TableColumn }[]): Promise<void>
```

* `table` - Table object or name
* `changedColumns` - array of changed columns.
  * `oldColumn` - old TableColumn object
  * `newColumn` - new TableColumn object

Changes a columns in the table.

```typescript
dropColumn(table: Table|string, column: TableColumn|string): Promise<void>
```

* `table` - Table object or name
* `column` - TableColumn object or column name to be dropped

Drops a column in the table.

```typescript
dropColumns(table: Table|string, columns: TableColumn[]): Promise<void>
```

* `table` - Table object or name
* `columns` - array of TableColumn objects to be dropped

Drops a columns in the table.

```typescript
createPrimaryKey(table: Table|string, columnNames: string[]): Promise<void>
```

* `table` - Table object or name
* `columnNames` - array of column names which will be primary

Creates a new primary key.

```typescript
updatePrimaryKeys(table: Table|string, columns: TableColumn[]): Promise<void>
```

* `table` - Table object or name
* `columns` - array of TableColumn objects which will be updated

Updates composite primary keys.

```typescript
dropPrimaryKey(table: Table|string): Promise<void>
```

* `table` - Table object or name

Drops a primary key.

```typescript
createUniqueConstraint(table: Table|string, uniqueConstraint: TableUnique): Promise<void>
```

* `table` - Table object or name
* `uniqueConstraint` - TableUnique object to be created

Creates new unique constraint.

> Note: does not work for MySQL, because MySQL stores unique constraints as unique indices. Use `createIndex()` method instead.

```typescript
createUniqueConstraints(table: Table|string, uniqueConstraints: TableUnique[]): Promise<void>
```

* `table` - Table object or name
* `uniqueConstraints` - array of TableUnique objects to be created

Creates new unique constraints.

> Note: does not work for MySQL, because MySQL stores unique constraints as unique indices. Use `createIndices()` method instead.

```typescript
dropUniqueConstraint(table: Table|string, uniqueOrName: TableUnique|string): Promise<void>
```

* `table` - Table object or name
* `uniqueOrName` - TableUnique object or unique constraint name to be dropped

Drops an unique constraint.

> Note: does not work for MySQL, because MySQL stores unique constraints as unique indices. Use `dropIndex()` method instead.

```typescript
dropUniqueConstraints(table: Table|string, uniqueConstraints: TableUnique[]): Promise<void>
```

* `table` - Table object or name
* `uniqueConstraints` - array of TableUnique objects to be dropped

Drops an unique constraints.

> Note: does not work for MySQL, because MySQL stores unique constraints as unique indices. Use `dropIndices()` method instead.

```typescript
createCheckConstraint(table: Table|string, checkConstraint: TableCheck): Promise<void>
```

* `table` - Table object or name
* `checkConstraint` - TableCheck object

Creates new check constraint.

> Note: MySQL does not support check constraints.

```typescript
createCheckConstraints(table: Table|string, checkConstraints: TableCheck[]): Promise<void>
```

* `table` - Table object or name
* `checkConstraints` - array of TableCheck objects

Creates new check constraint.

> Note: MySQL does not support check constraints.

```typescript
dropCheckConstraint(table: Table|string, checkOrName: TableCheck|string): Promise<void>
```

* `table` - Table object or name
* `checkOrName` - TableCheck object or check constraint name

Drops check constraint.

> Note: MySQL does not support check constraints.

```typescript
dropCheckConstraints(table: Table|string, checkConstraints: TableCheck[]): Promise<void>
```

* `table` - Table object or name
* `checkConstraints` - array of TableCheck objects

Drops check constraints.

> Note: MySQL does not support check constraints.

```typescript
createForeignKey(table: Table|string, foreignKey: TableForeignKey): Promise<void>
```

* `table` - Table object or name
* `foreignKey` - TableForeignKey object

Creates a new foreign key.

```typescript
createForeignKeys(table: Table|string, foreignKeys: TableForeignKey[]): Promise<void>
```

* `table` - Table object or name
* `foreignKeys` - array of TableForeignKey objects

Creates a new foreign keys.

```typescript
dropForeignKey(table: Table|string, foreignKeyOrName: TableForeignKey|string): Promise<void>
```

* `table` - Table object or name
* `foreignKeyOrName` - TableForeignKey object or foreign key name

Drops a foreign key.

```typescript
dropForeignKeys(table: Table|string, foreignKeys: TableForeignKey[]): Promise<void>
```

* `table` - Table object or name
* `foreignKeys` - array of TableForeignKey objects

Drops a foreign keys.

```typescript
createIndex(table: Table|string, index: TableIndex): Promise<void>
```

* `table` - Table object or name
* `index` - TableIndex object

Creates a new index.

```typescript
createIndices(table: Table|string, indices: TableIndex[]): Promise<void>
```

* `table` - Table object or name
* `indices` - array of TableIndex objects

Creates a new indices.

```typescript
dropIndex(table: Table|string, index: TableIndex|string): Promise<void>
```

* `table` - Table object or name
* `index` - TableIndex object or index name

Drops an index.

```typescript
dropIndices(table: Table|string, indices: TableIndex[]): Promise<void>
```

* `table` - Table object or name
* `indices` - array of TableIndex objects

Drops an indices.

```typescript
clearTable(tableName: string): Promise<void>
```

* `tableName` - table name

Clears all table contents.

> Note: this operation uses SQL's TRUNCATE query which cannot be reverted in transactions.

```typescript
enableSqlMemory(): void
```

Enables special query runner mode in which sql queries won't be executed, instead they will be memorized into a special variable inside query runner. You can get memorized sql using `getMemorySql()` method.

```typescript
disableSqlMemory(): void
```

Disables special query runner mode in which sql queries won't be executed. Previously memorized sql will be flushed.

```typescript
clearSqlMemory(): void
```

Flushes all memorized sql statements.

```typescript
getMemorySql(): SqlInMemory
```

* returns `SqlInMemory` object with array of `upQueries` and `downQueries` sql statements

Gets sql stored in the memory. Parameters in the sql are already replaced.

```typescript
executeMemoryUpSql(): Promise<void>
```

Executes memorized up sql queries.

```typescript
executeMemoryDownSql(): Promise<void>
```

Executes memorized down sql queries.

