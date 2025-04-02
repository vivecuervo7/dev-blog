---
title: Harnessing database-driven code generation
description: Use database introspection and code generation to enhance workflows and reflect schema changes across your entire codebase
slug: database-driven-codegen
date: 2025-03-25 00:00:00+0000
image: cover.jpg
categories:
  - Development
tags:
  - Postgres
  - SQL
  - Typescript
  - Kysely
  - Codegen
  - Zod
weight: 1
links:
  - title: graphile-migrate
    description: Opinionated SQL-powered productive roll-forward migration tool for PostgreSQL
    website: https://github.com/graphile/migrate
  - title: Kysely
    description: The type-safe SQL query builder for TypeScript
    website: https://kysely.dev/
  - title: kysely-codegen
    description: Generates Kysely type definitions from your database
    website: https://github.com/RobinBlomberg/kysely-codegen
  - title: Zod
    description: TypeScript-first schema validation with static type inference
    website: https://zod.dev/
---

As I started to really dive into SvelteKit, I couldn't help but find myself drawn to this idea of writing as little code as possible. Don't get me wrong &mdash; I still love working with C#, and honestly it's still where I'd turn to in a heartbeat if I had need to model a complex business domain.

I haven't been able to shake the thought though, that quite often we end up building rather small applications that might be simpler and faster to write using one of the current full-stack meta-frameworks. While SvelteKit has been my poison of choice in that regard, what is outlined here is framework-agnostic and should be applicable to any of the current offerings.

## ORMs, script builders, raw SQL...

I have to admit to feeling a little lost when I first stepped into the idea of managing persistence without my trusty Entity Framework Core. Instinctively, I reached for an ORM and did some poking around.

I came across [this fantastic post](https://sveltekit.io/blog/drizzle-sveltekit-integration) outlining the usage of [Drizzle](https://orm.drizzle.team/). I was pretty excited, things looked nice and easy to implement. Getting started was fairly straightforward using a Postgres database, but as I looked around I started stumbling across discussions such as [this one](https://github.com/thetutlage/meta/discussions/8) outlining the shortcomings of Drizzle.

Nice as it was, I decided to keep looking.

- [TypeORM](https://typeorm.io/) looked like a long term player, but seemed to be an unpopular choice.

- [Prisma](https://www.prisma.io/) looked a bit more positive, but I mistook their "pricing" page to indicate it was a paid option, which I later discovered was not the case, however the ship had sailed by this point.

- [MikroORM](https://mikro-orm.io/) received the most praise based on my research, but wasn't quite as easy to get set up as Drizzle.

By this point I was pretty over the idea of using an ORM in TypeScript-land.

### Or something else...?

I have to credit [this post on Reddit](https://www.reddit.com/r/sveltejs/comments/1brdr07/comment/kx9ymr9/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button) for pointing me in the direction I eventually found myself going in.

While the post itself goes into using GraphQL which isn't something I was particularly interested in, it did cause me to start looking into what a solution might look like if it were _truly_ data-driven &mdash; allowing your database to be the source of truth for your application.

This resonated with me, as someone who typically preferred a data-first approach. It aligned very closely to my typical workflow in .NET where I'd use [DbUp](https://dbup.readthedocs.io/en/latest/) to handle database migrations.

## The magic sauce

After a bit of a dig at just writing plain SQL, I decided to finally give [Kysely](https://kysely.dev/) a crack &mdash; a simple, no-nonsense query builder for TypeScript. It had consistently received high praise wherever I had seen it mentioned, and suffice to say I was _not_ disappointed when I finally used it.

But first, we need some data!

### graphile-migrate

The aforementioned Reddit post eventually landed me on [graphile-migrate](https://github.com/graphile/migrate) &mdash; a migration tool that introduced an interesting way to manage migrations.

While I can certainly envision some scenarios that could get a little messy, the premise is that you would write your migrations to be idempotent. Writing your migrations to a `current.sql` file would allow you to use the tool's `watch` mode, re-running the migration every time the file is changed.

Once work on the "current" migration is complete it can be committed, which will move it into the committed folder as a migration ready to be run against a production database.

Despite needing some additional thought to how best to write idempotent migrations, I found that in practice it allowed for a much more iterative workflow when compared to the usual sitting down and figuring out all of the data requirements as the very first step.

We'll use the following migration to illustrate how the other tools are used in a basic real-world scenario.

{{< code-hint current.sql >}}

```sql
drop table if exists app_user cascade;
drop table if exists league cascade;
drop table if exists player;
drop type if exists player_role;

create type player_role as enum ('batter', 'bowler', 'keeper', 'allrounder');

create table app_user (
    id uuid primary key
);

create table league (
    id uuid primary key default gen_random_uuid(),
    name varchar(30) not null unique,
    description varchar(200) not null,
    owner_id uuid references app_user not null
);

create table player (
    id uuid primary key default gen_random_uuid(),
    league_id uuid references league on delete cascade not null,
    name varchar(30) not null,
    role player_role not null,
    image_url varchar(2048)
);
```

### kysely-codegen

Before we can work with [Kysely](https://kysely.dev/), we need to provide a database schema in a format that Kysely can understand. Database introspection isn't something I'd done much of, and fortunately the Kysely documentation outlines a few tools that can aid with this.

[kysely-codegen](https://github.com/RobinBlomberg/kysely-codegen) as one of their offering purports to work with all dialects supported by Kysely itself, so that was the one that I picked off the list.

Configuration for `graphile-migrate` allows for running commands at various points, such as after all migrations, or after just the current migration. This allowed for a simple way to run `kysely-codegen` every time `graphile-migrate`'s `watch` function caused the migrations to re-run. Nice!

Running `kysely-codegen` against the database after the migration above was applied yields the following.

{{< code-hint db-schema.ts >}}

```typescript
/**
 * This file was generated by kysely-codegen.
 * Please do not edit it manually.
 */

import type { ColumnType } from "kysely";

export type Generated<T> = T extends ColumnType<infer S, infer I, infer U>
  ? ColumnType<S, I | undefined, U>
  : ColumnType<T, T | undefined, T>;

export type PlayerRole = "allrounder" | "batter" | "bowler" | "keeper";

export interface AppUser {
  id: string;
}

export interface League {
  description: string;
  id: Generated<string>;
  name: string;
  owner_id: string;
}

export interface Player {
  id: Generated<string>;
  image_url: string | null;
  league_id: string;
  name: string;
  role: PlayerRole;
}

export interface DB {
  app_user: AppUser;
  league: League;
  player: Player;
}
```

### kysely

I can't stress just how impressed I am with [Kysely](https://kysely.dev/) despite not having worked with it for long. It really does a fantastic job of not trying to do too much &mdash; it just gets out of the way and makes it feel quick and easy to work with.

Setting up the database client using Kysely was very straightforward.

{{< code-hint db.ts >}}

```typescript
const pool = new pg.Pool({ connectionString });
const dialect = new PostgresDialect({ pool });
const client = new Kysely<DB>({ dialect, log: ["error"] });
```

I decided to break the tables down into repositories for ease of access, and here we can see how simple queries look using Kysely. We get full type-safety along with auto-complete while writing the entire query, and I found that the query itself mapped nicely to what I'd expect the generated SQL to look like.

{{< code-hint players.ts >}}

```typescript
export class PlayerRepository extends RepositoryBase {
  public async create(player: Insertable<Player>) {
    return await this.client
      .insertInto("player")
      .values(player)
      .returning("id")
      .executeTakeFirstOrThrow();
  }

  public async update(id: string, values: Updateable<Player>) {
    await this.client
      .updateTable("player")
      .set(values)
      .where("id", "=", id)
      .execute();
  }

  public async delete(id: string) {
    await this.client.deleteFrom("player").where("id", "=", id).execute();
  }
}
```

Kysely provides us with `Selectable<T>`, `Insertable<T>` and `Updateable<T>` wrappers for each table, which should give us the correct types for each respective operation.

Additionally, you can replace `execute()` with `compile()` or `explain()` to get an idea of what's being generated under the hood. See the following for a truncated example based on running `compile()` against the `create(...)` command above.

```json
{
  query: { ... },
  sql: 'insert into "player" ("name", "image_url", "role", "league_id") values ($1, $2, $3, $4) returning "id"',
  parameters: [
    'Isaac Dedini',
    '<image_url>',
    'bowler',
    '<uuid>'
  ]
}
```

## Going even further

I was pretty happy with what I had landed on here, but I started wondering if we might be able to also extract some additional metadata and generate validation schema based on some of the database properties &mdash; namely wherever we've used `varchar(n)`.

Yeah, as you can probably tell I've run into a few cases where our application's validation hasn't been updated to reflect a change in the database schema. I wanted to try and fix that.

I started out here looking around at various options, including replacing `kysely-codegen` with [kanel-kysely](https://github.com/kristiandupont/kanel/tree/main/packages/kanel-kysely) just so I could leverage [kanel-zod](https://www.npmjs.com/package/kanel-zod) for this purpose. I was left underwhelmed at what was actually generated, and decided to go back to the drawing board.

### Generating Zod validation schema with kysely-codegen

The [0.18.0 release](https://github.com/RobinBlomberg/kysely-codegen/releases/tag/0.18.0) of `kysely-codegen` introduced the ability to define custom serializers, largely related to this [GitHub discussion](https://github.com/RobinBlomberg/kysely-codegen/issues/86). This gave me _most_ of what I was after once I'd extended the example provided in the linked release notes to include more extensive mappings.

Something that appeared to be missing yet as one of the key reasons I wanted to look into database-driven Zod schema was the ability to extract maximum character lengths for `varchar` columns. Of course, [there is advice floating](https://wiki.postgresql.org/wiki/Don't_Do_This#Don.27t_use_varchar.28n.29_by_default) around that suggests preferring `text` instead of `varchar`, but it's a habit I can't shake.

I also wanted to refrain from mapping check constraints &mdash; I needed to draw a line here and stopping short of check constraints felt like a good balance between simplicity and functionality. So, consider that to be a caveat of this approach, although nothing is stopping you from extending the serializer to account for check constraints either.

#### Scripts

I eventually settled on adding the following script to allow me to grab additional metadata from the database, essentially returning a mapping between column names and their `character_maximum_length` property.

{{< code-hint get-column-metadata.js >}}

```typescript
import pg from "pg";

const table = process.argv[2];

const client = new pg.Client({
  connectionString: process.env.DATABASE_URL || "",
});
await client.connect();

const query = `
  SELECT column_name, character_maximum_length
  FROM information_schema.columns 
  WHERE table_name = $1;
`;
const result = await client.query(query, [table]);

await client.end();

console.log(JSON.stringify(result.rows));
```

#### Configuration

I've truncated the `mapDataTypeToZodString(...)` function below for brevity, but it retains all the mappings required for the tables defined so far.

{{< code-hint ".kysely-zod-codegenrc.ts" >}}

```typescript
import "dotenv/config";
import { toKyselyCamelCase } from "kysely-codegen";
import { join } from "path";
import { execSync } from "child_process";

export default {
  url: process.env.DATABASE_URL || "",
  outFile: join(process.cwd(), "src", "lib", "generated", "db-schema-zod.ts"),
  excludePattern: "graphile_migrate.*",
  serializer: {
    serializeFile: (metadata) => {
      let output = 'import { z } from "zod";\n\n';

      for (const table of metadata.tables) {
        output += "export const ";
        output += toKyselyCamelCase(table.name);
        output += "Schema = z.object({\n";

        const additionalColumnMetadata = getAdditionalColumnMetadata(
          table.name
        );

        for (const column of table.columns) {
          output += "  ";
          output += column.name;
          output += ": ";

          output += mapDataTypeToZodString(column.dataType, column.enumValues);

          if (
            !column.isNullable &&
            (column.dataType === "text" ||
              column.dataType === "character varying" ||
              column.dataType === "varchar")
          ) {
            output += ".min(1)";
          }

          const maxLength = additionalColumnMetadata.find(
            (c) => c.column_name === column.name
          )?.character_maximum_length;
          if (maxLength) {
            output += `.max(${maxLength})`;
          }

          if (column.isNullable) {
            output += ".optional()";
          }

          output += ",\n";
        }

        output += "});\n\n";
      }

      return output;
    },
  },
};

function getAdditionalColumnMetadata(tableName: string) {
  const result = execSync(
    `node scripts/get-column-metadata.js "${tableName}"`,
    { encoding: "utf-8" }
  );
  return JSON.parse(result);
}

function mapDataTypeToZodString(dataType, enumValues) {
  switch (dataType) {
    case "integer":
      return "z.number()";

    case "varchar":
      return "z.string()";

    case "uuid":
      return "z.string().uuid()";

    default:
      return enumValues && enumValues.length > 0
        ? `z.enum([${enumValues.map((value) => `'${value}'`).join(", ")}])`
        : "z.unknown()";
  }
}
```

#### Output

{{< code-hint db-schema-zod.ts >}}

```typescript
import { z } from "zod";

export const appUserSchema = z.object({
  id: z.string().uuid(),
});

export const appUserLeagueSchema = z.object({
  user_id: z.string().uuid(),
  league_id: z.string().uuid(),
});

export const leagueSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(30),
  description: z.string().min(1).max(200),
  owner_id: z.string().uuid(),
});

export const memberSchema = z.object({
  id: z.string().uuid(),
  league_id: z.string().uuid(),
  name: z.string().min(1).max(30),
  role: z.string().min(1).max(30),
});

export const playerSchema = z.object({
  id: z.string().uuid(),
  league_id: z.string().uuid(),
  name: z.string().min(1).max(30),
  role: z.enum(["allrounder", "batter", "bowler", "keeper"]),
  image_url: z.string().max(2048).optional(),
});

export const roundSchema = z.object({
  id: z.string().uuid(),
  league_id: z.string().uuid(),
});
```

#### Usage

Now that we have our Zod validation schemas, we can use them for form validation. In the case of a create player form we only ask the user to supply three of the properties, so we just restrict the schema accordingly.

Once that's done we can extend the schema to _add_ rules to an individual property. This allows us to layer business rules on top of our database-driven validation.

{{< code-hint db-schema-zod.ts >}}

```typescript
import { playerSchema } from "$lib/generated/db-schema-zod";
import { z } from "zod";

export const formSchema = playerSchema
  .pick({ name: true, image_url: true, role: true })
  .extend({
    image_url: playerSchema.shape.image_url.unwrap().url().or(z.literal("")),
  });
```

In this case, changing `player.image_url` from `varchar(2048)` to `varchar(20)` would cause the form's validation to instantly reflect that change without needing us to go and modify a separate schema.

## Conclusion

After digging through a swathe of options when it comes to managing persistence in a full-stack meta-framework, eventually stumbling upon the above felt like a revelation.

While no doubt there will be hitches along the way, this presents a way to move forward quickly and iteratively with data changes, allowing them to propagate throughout your codebase via the generated types.

Essentially, this boils down to a very straightforward workflow &mdash; add a migration, let it spin for a second or two, and start using the generated types.
