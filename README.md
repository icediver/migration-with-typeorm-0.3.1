# [Database migrations with TypeORM](https://wanago.io/2022/07/25/api-nestjs-database-migrations-typeorm/)

When working with relational databases, we define the structure of the data rather strictly. For example, we need to
specify the format of every table along with fields, relations, indexes, and other structures. By doing that, we also
tell the database how to validate the incoming data.

It is crucial to think about the structure of our database carefully. Even if we do that, the requirements that our
application has to meet change. Because of the above, we rarely can avoid having to modify the structure of our
database. When doing that, we need to be careful not to lose any existing data.

With database migrations, we can define a set of controlled changes that aim to modify the structure of the data. They
can include adding or removing tables, changing columns, or changing the data types, for example. While we could
manually run SQL queries that make the necessary adjustments, this is not the optimal approach. Instead, we want our
migrations to be easy to repeat across different application environments.

Also, we need to acknowledge that modifying the structure of the database is a delicate process where things can go
wrong and damage the existing data. Fortunately, writing database migrations includes committing them to the repository.
Therefore, they can undergo a rigorous review before merging to the master branch. In this article, we go through the
idea of migrations and learn how to perform them with TypeORM.

Working with migrations using TypeORM
When configuring TypeORM, we can set the synchronize property to true. This causes TypeORM to synchronize the database
with our entities automatically. However, using it in production is highly discouraged because it might lead to
unexpected data loss.

Instead, TypeORM has a tool that helps us create and run migrations. Unfortunately, its migration documentation is
outdated and does not match the latest version.

# Configuring the TypeORM CLI

To start working with migrations using TypeORM, we need to properly configure its command line interface (CLI). To do
that, we need to create a designated configuration file.

__`typeOrm.config.ts`__

```typescript
import { DataSource } from 'typeorm';
import { ConfigService } from '@nestjs/config';
import { config } from 'dotenv';

config();

const configService = new ConfigService();

export default new DataSource({
type: 'postgres',
host: configService.get('POSTGRES_HOST'),
port: configService.get('POSTGRES_PORT'),
username: configService.get('POSTGRES_USER'),
password: configService.get('POSTGRES_PASSWORD'),
database: configService.get('POSTGRES_DB'),
entities: [],
});
```

> Above, we use dotenv to make sure the ConfigService loaded the environment variables before using it.

We also need to add some entries to the scripts in our `package.json`.

`package.json`

```json
"scripts": {
"typeorm": "ts-node ./node_modules/typeorm/cli",
"typeorm:run-migrations": "npm run typeorm migration:run -- -d ./typeOrm.config.ts",
"typeorm:generate-migration": "npm run typeorm -- -d ./typeOrm.config.ts migration:generate ./migrations/$npm_config_name",
"typeorm:create-migration": "npm run typeorm -- migration:create ./migrations/$npm_config_name",
"typeorm:revert-migration": "npm run typeorm -- -d ./typeOrm.config.ts migration:revert",
...
}
```

> Unfortunately, the __`$npm_config`__ feature is not supported by yarn.

## Generating our first migration

Let’s define a straightforward entity of a post.

`post.entity.ts`

```typescript
import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
class PostEntity {
@PrimaryGeneratedColumn('identity', {
generatedIdentity: 'ALWAYS',
})
id: number;

@Column()
title: string;

@Column()
content: string;
}

export default PostEntity;
```

> If you want to know more about identity columns, check out Serial type versus identity columns in PostgreSQL and
> TypeORM

There is a significant caveat regarding the entities directory in our configuration. Let’s take a look at our NestJS
database configuration.

`database.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
imports: [
TypeOrmModule.forRootAsync({
imports: [ConfigModule],
inject: [ConfigService],
useFactory: (configService: ConfigService) => ({
type: 'postgres',
host: configService.get('POSTGRES_HOST'),
port: configService.get('POSTGRES_PORT'),
username: configService.get('POSTGRES_USER'),
password: configService.get('POSTGRES_PASSWORD'),
database: configService.get('POSTGRES_DB'),
entities: [],
autoLoadEntities: true,
}),
}),
],
})
class DatabaseModule {}

export default DatabaseModule;

```

The @nestjs/typeorm library implements the `autoLoadEntities` that analyzes our NestJS application and identifies all of
our entities. Unfortunately, the basic TypeORM configuration can’t do that.

> We still need to add the entities we don’t use through `TypeOrmModule.forFeature()` to the entities array.

Because of the above, we need to manually add the `PostEntity` to our `entities` array in our CLI configuration.

`typeOrm.config.ts`

```typescript
import { DataSource } from 'typeorm';
import { ConfigService } from '@nestjs/config';
import { config } from 'dotenv';
import PostEntity from './src/posts/post.entity';

config();

const configService = new ConfigService();

export default new DataSource({
type: 'postgres',
host: configService.get('POSTGRES_HOST'),
port: configService.get('POSTGRES_PORT'),
username: configService.get('POSTGRES_USER'),
password: configService.get('POSTGRES_PASSWORD'),
database: configService.get('POSTGRES_DB'),
entities: [PostEntity],
});
```

> We might be able to figure out a better approach if the CLI
> would [support asynchronous DataSource creation](https://github.com/typeorm/typeorm/issues/8914). Once
> the
> [pull request with the improvement](https://github.com/typeorm/typeorm/pull/8917) is merged, we could create a NestJS
> application in our typeOrm.config.ts file
> and take
> advantage of the autoLoadEntities property.

Once we have all of the above set up, we can use the migration:generate command to let TypeORM generate the migration
file.

`npm run typeorm:generate-migration --name=CreatePost`

Running the migration command creates a file with the code that can bring our database from one state to another and
back. Its filename consists of the current timestamp followed by the name provided when using the migration:generate
command.

`1658694616973-CreatePost.ts`

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class CreatePost1658694616973 implements MigrationInterface {
name = 'CreatePost1658694616973';

public async up(queryRunner: QueryRunner): Promise<void> {
await queryRunner.query(
`CREATE TABLE "post_entity" ("id" integer GENERATED ALWAYS AS IDENTITY NOT NULL, "title" character varying NOT NULL, "content" character varying NOT NULL, CONSTRAINT "PK_58a149c4e88bf49036bc4c8c79f" PRIMARY KEY ("id"))`
,
);
}

public async down(queryRunner: QueryRunner): Promise<void> {
await queryRunner.query(`DROP TABLE "post_entity"`);
}
}
```

> I used prettier on the generated file.

Above, there are two methods:

`up` – performs the migration,
`down` – reverts it.

## Running the migrations

To run a migration, we must add it to the `migrations` array in our `typeOrm.config.ts` file. Unfortunately, using
strings
with the migrations array is deprecated and will stop working in TypeORM 0.4. Because of that, [we should import the
migration classes manually](https://github.com/typeorm/typeorm/issues/8762).

`typeOrm.config.ts`

```typescript
import { DataSource } from 'typeorm';
import { ConfigService } from '@nestjs/config';
import { config } from 'dotenv';
import PostEntity from './src/posts/post.entity';
import { CreatePost1658694616973 } from './migrations/1658694616973-CreatePost';

config();

const configService = new ConfigService();

export default new DataSource({
type: 'postgres',
host: configService.get('POSTGRES_HOST'),
port: configService.get('POSTGRES_PORT'),
username: configService.get('POSTGRES_USER'),
password: configService.get('POSTGRES_PASSWORD'),
database: configService.get('POSTGRES_DB'),
entities: [PostEntity],
migrations: [CreatePost1658694616973],
});
```

Once we have the `migration` added to the migrations array, we can run the command to execute it.

`npm run typeorm:run-migrations`

The above command yields the following logs:

```typescript
query: SELECT * FROM current_schema()
query: SHOW server_version;
query: SELECT * FROM "information_schema"."tables" WHERE "table_schema" = 'public' AND "table_name" = 'migrations'
query: SELECT * FROM "migrations" "migrations" ORDER BY "id" DESC
0 migrations are already loaded in the database.
1 migrations were found in the source code.
1 migrations are new migrations must be executed.
query: START TRANSACTION
query: CREATE TABLE "post_entity" ("id" integer GENERATED ALWAYS AS IDENTITY NOT NULL, "title" character varying NOT
NULL, "content" character varying NOT NULL, CONSTRAINT "PK_58a149c4e88bf49036bc4c8c79f" PRIMARY KEY ("id"))
query: INSERT INTO "migrations"("timestamp", "name") VALUES ($1, $2) --
PARAMETERS: [1658694616973,"CreatePost1658694616973"]
Migration CreatePost1658694616973 has been executed successfully.
query: COMMIT
```

Running the migration command does a few things. First, it identifies that the `migrations` array contains a migration
that wasn’t executed yet. It runs the `up` method and creates the posts table.

Besides that, it also adds an entry to the `migrations` table in the database. It indicates that the migration was
executed.

## Reverting migrations

To revert a migration, we need to use the migration:revert command.

`npm run typeorm:revert-migration`

The above command produces the following logs:

```bash
query: SELECT * FROM current_schema()
query: SHOW server_version;
query: SELECT * FROM "information_schema"."tables" WHERE "table_schema" = 'public' AND "table_name" = 'migrations'
query: SELECT * FROM "migrations" "migrations" ORDER BY "id" DESC
1 migrations are already loaded in the database.
CreatePost1658694616973 is the last executed migration. It was executed on Sun Jul 24 2022 22:30:16 GMT+0200 (Central
European Summer Time).
Now reverting it...
query: START TRANSACTION
query: DROP TABLE "post_entity"
query: DELETE FROM "migrations" WHERE "timestamp" = $1 AND "name" = $2 --
PARAMETERS: [1658694616973,"CreatePost1658694616973"]
Migration CreatePost1658694616973 has been reverted successfully.
query: COMMIT
```

Running the `revert` command executes the down method in the latest performed migration and removes the respective row
from the `migrations` array. Therefore, if we need to revert more than one migration, we must use the command multiple
times.

## Creating migrations manually

Besides relying on TypeORM to generate the migrations for us, we can write their logic manually. Let’s start by making a
slight change to the `PostEntity`.

`post.entity.ts`

```typescript
import {
Column,
CreateDateColumn,
Entity,
PrimaryGeneratedColumn,
} from 'typeorm';

@Entity()
class PostEntity {
@PrimaryGeneratedColumn('identity', {
generatedIdentity: 'ALWAYS',
})
id: number;

@Column()
title: string;

@Column()
content: string;

@CreateDateColumn({ type: 'timestamptz' })
createdAt: Date;
}

export default PostEntity;
```

If you want to know more about managing dates with PostgreSQL, check out [Managing date and time with PostgreSQL and
TypeORM](https://wanago.io/2021/03/15/postgresql-typeorm-date-time/)

Now, let’s run a command that tells TypeORM to create the basics of the migration for us.

`npm run typeorm:create-migration --name=PostCreationDate  `

By doing the above, we end up with the following file:

`1658701645714-PostCreationDate.ts`

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class PostCreationDate1658701645714 implements MigrationInterface {
public async up(queryRunner: QueryRunner): Promise<void> {}

public async down(queryRunner: QueryRunner): Promise<void> {}
}
```

> I used prettier on the generated file.

When generating migrations, TypeORM uses queryRunner.query and provides a raw SQL query. While that’s a viable approach,
we can
also [use the migration API](https://github.com/typeorm/typeorm/blob/master/docs/migrations.md#using-migration-api-to-write-migrations)
.

`1658701645714-PostCreationDate.ts`

```typescript
import { MigrationInterface, QueryRunner, TableColumn } from 'typeorm';

export class PostCreationDate1658701645714 implements MigrationInterface {
public async up(queryRunner: QueryRunner): Promise<void> {
await queryRunner.addColumn(
'post_entity',
new TableColumn({
name: 'createdAt',
type: 'timestamptz',
}),
);
}

public async down(queryRunner: QueryRunner): Promise<void> {
await queryRunner.dropColumn('post_entity', 'createdAt');
}
}
```

To run our migration, we also need to add it to the `migrations` array.

`typeOrm.config.ts`

```typescript
import { DataSource } from 'typeorm';
import { ConfigService } from '@nestjs/config';
import { config } from 'dotenv';
import PostEntity from './src/posts/post.entity';
import { CreatePost1658694616973 } from './migrations/1658694616973-CreatePost';
import { PostCreationDate1658701645714 } from './migrations/1658701645714-PostCreationDate';

config();

const configService = new ConfigService();

export default new DataSource({
type: 'postgres',
host: configService.get('POSTGRES_HOST'),
port: configService.get('POSTGRES_PORT'),
username: configService.get('POSTGRES_USER'),
password: configService.get('POSTGRES_PASSWORD'),
database: configService.get('POSTGRES_DB'),
entities: [PostEntity],
migrations: [CreatePost1658694616973, PostCreationDate1658701645714],
});
```

Now, we can execute the migration by running the appropriate command.

`npm run typeorm:run-migrations`

Running the command gives us the following logs:

> query: SELECT * FROM current_schema()
> query: SHOW server_version;
> query: SELECT * FROM "information_schema"."tables" WHERE "table_schema" = 'public' AND "table_name" = 'migrations'
> query: SELECT * FROM "migrations" "migrations" ORDER BY "id" DESC
> 1 migrations are already loaded in the database.
> 2 migrations were found in the source code.
> CreatePost1658694616973 is the last executed migration. It was executed on Sun Jul 24 2022 22:30:16 GMT+0200 (Central
> European Summer Time).
> 1 migrations are new migrations must be executed.
> query: START TRANSACTION
> query: SELECT * FROM current_schema()
> query: SELECT * FROM current_database()
> query: SELECT "table_schema", "table_name" FROM "information_schema"."tables" WHERE ("table_schema" = 'public' AND "
> table_name" = 'post_entity')
> query: SELECT TRUE FROM information_schema.columns WHERE table_name = 'pg_class' and column_name = 'relispartition'
> query: SELECT columns.*, pg_catalog.col_description(('"' || table_catalog || '"."' || table_schema || '"."' ||
> table_name || '"')::regclass::oid, ordinal_position) AS description, ('"' || "udt_schema" || '"."' || "udt_name"
> || '"')::"regtype" AS "regtype", pg_catalog.format_type("col_attr"."atttypid", "col_attr"."atttypmod") AS "format_type"
> FROM "information_schema"."columns" LEFT JOIN "pg_catalog"."pg_attribute" AS "col_attr" ON "col_attr"."attname" = "
> columns"."column_name" AND "col_attr"."attrelid" = ( SELECT "cls"."oid" FROM "pg_catalog"."pg_class" AS "cls" LEFT
> JOIN "pg_catalog"."pg_namespace" AS "ns" ON "ns"."oid" = "cls"."relnamespace" WHERE "cls"."relname" = "columns"."
> table_name" AND "ns"."nspname" = "columns"."table_schema" ) WHERE ("table_schema" = 'public' AND "table_name" = '
> post_entity')
> query: SELECT "ns"."nspname" AS "table_schema", "t"."relname" AS "table_name", "cnst"."conname" AS "constraint_name",
> pg_get_constraintdef("cnst"."oid") AS "expression", CASE "cnst"."contype" WHEN 'p' THEN 'PRIMARY' WHEN 'u' THEN 'UNIQUE'
> WHEN 'c' THEN 'CHECK' WHEN 'x' THEN 'EXCLUDE' END AS "constraint_type", "a"."attname" AS "column_name" FROM "
> pg_constraint" "cnst" INNER JOIN "pg_class" "t" ON "t"."oid" = "cnst"."conrelid" INNER JOIN "pg_namespace" "ns" ON "ns"
> ."oid" = "cnst"."connamespace" LEFT JOIN "pg_attribute" "a" ON "a"."attrelid" = "cnst"."conrelid" AND "a"."attnum" =
> ANY ("cnst"."conkey") WHERE "t"."relkind" IN ('r', 'p') AND (("ns"."nspname" = 'public' AND "t"."relname" = '
> post_entity'))
> query: SELECT "ns"."nspname" AS "table_schema", "t"."relname" AS "table_name", "i"."relname" AS "constraint_name", "a"."
> attname" AS "column_name", CASE "ix"."indisunique" WHEN 't' THEN 'TRUE' ELSE'FALSE' END AS "is_unique", pg_get_expr("ix"
> ."indpred", "ix"."indrelid") AS "condition", "types"."typname" AS "type_name" FROM "pg_class" "t" INNER JOIN "
> pg_index" "ix" ON "ix"."indrelid" = "t"."oid" INNER JOIN "pg_attribute" "a" ON "a"."attrelid" = "t"."oid"  AND "a"."
> attnum" = ANY ("ix"."indkey") INNER JOIN "pg_namespace" "ns" ON "ns"."oid" = "t"."relnamespace" INNER JOIN "pg_class" "
> i" ON "i"."oid" = "ix"."indexrelid" INNER JOIN "pg_type" "types" ON "types"."oid" = "a"."atttypid" LEFT JOIN "
> pg_constraint" "cnst" ON "cnst"."conname" = "i"."relname" WHERE "t"."relkind" IN ('r', 'p') AND "cnst"."contype" IS NULL
> AND (("ns"."nspname" = 'public' AND "t"."relname" = 'post_entity'))
> query: SELECT "con"."conname" AS "constraint_name", "con"."nspname" AS "table_schema", "con"."relname" AS "table_name"
> , "att2"."attname" AS "column_name", "ns"."nspname" AS "referenced_table_schema", "cl"."relname" AS "
> referenced_table_name", "att"."attname" AS "referenced_column_name", "con"."confdeltype" AS "on_delete", "con"."
> confupdtype" AS "on_update", "con"."condeferrable" AS "deferrable", "con"."condeferred" AS "deferred" FROM ( SELECT
> UNNEST ("con1"."conkey") AS "parent", UNNEST ("con1"."confkey") AS "child", "con1"."confrelid", "con1"."conrelid", "
> con1"."conname", "con1"."contype", "ns"."nspname", "cl"."relname", "con1"."condeferrable", CASE WHEN "con1"."
> condeferred" THEN 'INITIALLY DEFERRED' ELSE 'INITIALLY IMMEDIATE' END as condeferred, CASE "con1"."confdeltype" WHEN 'a'
> THEN 'NO ACTION' WHEN 'r' THEN 'RESTRICT' WHEN 'c' THEN 'CASCADE' WHEN 'n' THEN 'SET NULL' WHEN 'd' THEN 'SET DEFAULT'
> END as "confdeltype", CASE "con1"."confupdtype" WHEN 'a' THEN 'NO ACTION' WHEN 'r' THEN 'RESTRICT' WHEN 'c' THEN '
> CASCADE' WHEN 'n' THEN 'SET NULL' WHEN 'd' THEN 'SET DEFAULT' END as "confupdtype" FROM "pg_class" "cl" INNER JOIN "
> pg_namespace" "ns" ON "cl"."relnamespace" = "ns"."oid" INNER JOIN "pg_constraint" "con1" ON "con1"."conrelid" = "cl"."
> oid" WHERE "con1"."contype" = 'f' AND (("ns"."nspname" = 'public' AND "cl"."relname" = 'post_entity')) ) "con" INNER
> JOIN "pg_attribute" "att" ON "att"."attrelid" = "con"."confrelid" AND "att"."attnum" = "con"."child" INNER JOIN "
> pg_class" "cl" ON "cl"."oid" = "con"."confrelid"  AND "cl"."relispartition" = 'f'INNER JOIN "pg_namespace" "ns" ON "cl"
> ."relnamespace" = "ns"."oid" INNER JOIN "pg_attribute" "att2" ON "att2"."attrelid" = "con"."conrelid" AND "att2"."
> attnum" = "con"."parent"
> query: ALTER TABLE "post_entity" ADD "createdAt" timestamptz NOT NULL
> query: INSERT INTO "migrations"("timestamp", "name") VALUES ($1, $2) --
> PARAMETERS: [1658701645714,"PostCreationDate1658701645714"]
> Migration PostCreationDate1658701645714 has been executed successfully.
> query: COMMIT
>
>
>We can notice that using the migrations API seems to produce bigger queries when executing the migrations.

If we run our new migration, the `migrations` table gets a new row:

## Summary

In this article, we’ve learned what migrations are and how to manage them with TypeORM. To do that, we’ve set up the
TypeORM CLI and used commands for creating, generating, running, and reverting migrations. We’ve also used the
migrations API and noticed that it produces rather big SQL queries. All of the above gave us a thorough understanding of
how to deal with migrations using TypeOrm.

