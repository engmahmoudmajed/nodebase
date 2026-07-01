# Prisma + Next.js: Complete Guide

A full reference for using Prisma ORM inside a Next.js application — from installation to production-ready patterns.

---

## 1. What Is Prisma?

Prisma is a type-safe, next-generation ORM for Node.js and TypeScript. It has three main pieces:

- **Prisma Client** — an auto-generated, type-safe query builder used in your app code.
- **Prisma Migrate** — a declarative migration system that turns schema changes into SQL.
- **Prisma Studio** — a local GUI to browse and edit your database data.

Why it pairs well with Next.js:
- Full TypeScript type inference for every model/query.
- Works in Route Handlers, Server Components, Server Actions, and API Routes.
- Driver adapters make it compatible with serverless/edge deployments (Vercel, Cloudflare).

---

## 2. Installing Prisma in a Next.js Project

```bash
npm install prisma --save-dev
npm install @prisma/client
npx prisma init
```

`npx prisma init` creates:
- `prisma/schema.prisma` — your schema definition file.
- `.env` — where `DATABASE_URL` lives.

Add your real connection string in `.env`:

```env
DATABASE_URL="postgresql://user:password@localhost:5432/mydb?schema=public"
```

---

## 3. The `schema.prisma` File

Three sections:

```prisma
generator client {
  provider = "prisma-client-js"
  output   = "./generated/prisma/client" // optional custom output path
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
  posts Post[]
}

model Post {
  id        Int     @id @default(autoincrement())
  title     String
  published Boolean @default(false)
  authorId  Int
  author    User    @relation(fields: [authorId], references: [id])
}
```

- **generator** — what client code to generate, and where.
- **datasource** — which database engine and connection string to use.
- **model** — maps to a table (SQL) or collection (MongoDB). Relations (`@relation`), unique constraints (`@unique`), defaults (`@default`), and IDs (`@id`) are all declared here.

---

## 4. `generate` vs `migrate dev` — Why You Can't Just Run `generate`

This is a common point of confusion, so it's worth being explicit about it.

Prisma manages **two separate things** that need to stay in sync:

1. **The actual database** — the real tables, columns, and types living in Postgres/MySQL/etc.
2. **The Prisma Client code** — the TypeScript types and query methods your app imports.

- `npx prisma generate` only touches #2. It **never talks to your database**. It just reads `schema.prisma` and regenerates the TypeScript client so your code has autocomplete/types matching whatever is written in the schema file.
- `npx prisma migrate dev` touches **both** — it changes the real database *and* then calls `generate` for you automatically.

### Why "just running generate" causes problems

Say your schema currently has:

```prisma
model User {
  id    Int    @id @default(autoincrement())
  email String
}
```

And your database table `User` already exists with just `id, email`.

Now you add a field:

```prisma
model User {
  id    Int    @id @default(autoincrement())
  email String
  age   Int
}
```

If you run `npx prisma generate`:
- TypeScript will now think `User` has an `age` field (the client is updated).
- But the actual database table **still only has `id, email`** — nothing changed there.
- The moment you try `prisma.user.create({ data: { email: "x", age: 20 } })`, it will throw a runtime error like `Column "age" does not exist` — because the client is *lying* to TypeScript about what the DB actually looks like.

So `generate` alone gets your **types out of sync with reality**. It compiles fine, looks fine in your editor, and then blows up at runtime.

### What `migrate dev` actually does, step by step

```bash
npx prisma migrate dev --name add_age_to_user
```

1. Compares your `schema.prisma` to the current DB state.
2. Writes a SQL file, e.g. `prisma/migrations/20260630_add_age_to_user/migration.sql`, containing something like:
   ```sql
   ALTER TABLE "User" ADD COLUMN "age" INTEGER NOT NULL;
   ```
3. **Runs that SQL against your dev database** — so the table physically now has the `age` column.
4. Runs `generate` for you, so the client types now match the (now-updated) database.

That SQL migration file also gets committed to git, so anyone else on the project (or your production deploy) can replay the exact same change with `prisma migrate deploy`.

### Simple mental model

| Command | Changes the database? | Changes the TS client? |
|---|---|---|
| `npx prisma generate` | ❌ No | ✅ Yes |
| `npx prisma migrate dev` | ✅ Yes | ✅ Yes (calls generate internally) |
| `npx prisma db push` | ✅ Yes (no migration file saved) | ✅ Yes |
| `npx prisma migrate deploy` | ✅ Yes (applies existing migration files) | ❌ No — run `generate` separately if needed |

**Rule of thumb:** whenever you edit `schema.prisma` and want that change to exist in your actual database, you need `migrate dev` (or `db push` for quick prototyping without migration history). `generate` by itself is only for when the schema and DB are already in sync and you just need the client rebuilt — e.g. after `git pull` when a teammate already ran the migration and committed the SQL file, and you just need to apply it locally and regenerate types.

---

## 5. Migrations: Applying Schema to the Database

Every time you edit `schema.prisma`, run:

```bash
npx prisma migrate dev --name describe_the_change
```

This:
1. Diffs your schema against the database.
2. Generates a SQL migration file under `prisma/migrations/`.
3. Applies it to your dev database.
4. Regenerates the Prisma Client automatically.

For production deployments (CI/CD), use the non-interactive command instead:

```bash
npx prisma migrate deploy
```

If you already have an existing database and want Prisma to infer the schema from it:

```bash
npx prisma db pull
```

---

## 6. Generating the Client

`migrate dev` runs this for you, but you'll need it manually whenever:
- You pull new code from teammates with schema changes.
- You edit the schema without running a migration (e.g. `prisma db push` for prototyping).

```bash
npx prisma generate
```

---

## 7. Driver Adapters (Edge / Serverless Support)

By default, Prisma Client uses its own Rust query engine binary, which doesn't run in edge runtimes. **Driver adapters** let Prisma use a native JS database driver instead, making it compatible with serverless and edge functions.

```bash
npm install @prisma/adapter-pg pg
```

```ts
import { PrismaPg } from "@prisma/adapter-pg";

const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL });
const prisma = new PrismaClient({ adapter });
```

Enable it in the schema:

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["driverAdapters"]
}
```

Common adapters:
| Database | Adapter package |
|---|---|
| PostgreSQL | `@prisma/adapter-pg` |
| MySQL | `@prisma/adapter-mysql` |
| SQLite / Turso | `@prisma/adapter-libsql` |
| Neon (serverless Postgres) | `@prisma/adapter-neon` |
| PlanetScale | `@prisma/adapter-planetscale` |

---

## 8. The Singleton Pattern (Critical for Next.js)

Next.js hot-reloads your code on every save in development. Without care, this spawns a **new** Prisma Client (and new DB connections) on every reload, quickly exhausting your database's connection limit.

**`lib/prisma.ts`:**

```ts
import { PrismaClient } from "./generated/prisma/client";
import { PrismaPg } from "@prisma/adapter-pg";

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL });

const prisma = globalForPrisma.prisma ?? new PrismaClient({ adapter });

if (process.env.NODE_ENV !== "production") {
  globalForPrisma.prisma = prisma;
}

export default prisma;
```

**How it works:**
1. `globalThis` is cast to a type that allows a `prisma` property (TypeScript won't allow attaching arbitrary properties otherwise).
2. `??` reuses the existing global instance if one exists, or creates a new one.
3. In development only, the instance is cached on `globalThis` so the next hot-reload finds it instead of creating a duplicate connection.
4. In production, each serverless function instance only initializes once anyway, so this caching step is skipped.

Import it anywhere in your app:

```ts
import prisma from "@/lib/prisma";
```

---

## 9. Using Prisma Client in Next.js (App Router)

### Server Component (direct DB read, no API layer needed)

```tsx
import prisma from "@/lib/prisma";

export default async function UsersPage() {
  const users = await prisma.user.findMany({
    include: { posts: true },
  });

  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>{u.name} — {u.posts.length} posts</li>
      ))}
    </ul>
  );
}
```

### Server Action (mutations)

```ts
"use server";

import prisma from "@/lib/prisma";
import { revalidatePath } from "next/cache";

export async function createUser(formData: FormData) {
  await prisma.user.create({
    data: {
      email: formData.get("email") as string,
      name: formData.get("name") as string,
    },
  });
  revalidatePath("/users");
}
```

### Route Handler (`app/api/users/route.ts`)

```ts
import prisma from "@/lib/prisma";
import { NextResponse } from "next/server";

export async function GET() {
  const users = await prisma.user.findMany();
  return NextResponse.json(users);
}

export async function POST(request: Request) {
  const body = await request.json();
  const user = await prisma.user.create({ data: body });
  return NextResponse.json(user, { status: 201 });
}
```

---

## 10. Common Query Patterns

```ts
// Find one
await prisma.user.findUnique({ where: { id: 1 } });

// Find with filters
await prisma.post.findMany({ where: { published: true } });

// Create with nested relation
await prisma.user.create({
  data: {
    email: "a@b.com",
    posts: { create: [{ title: "Hello world" }] },
  },
});

// Update
await prisma.user.update({
  where: { id: 1 },
  data: { name: "New Name" },
});

// Delete
await prisma.user.delete({ where: { id: 1 } });

// Transaction (atomic multi-query)
await prisma.$transaction([
  prisma.user.update({ where: { id: 1 }, data: { name: "A" } }),
  prisma.post.deleteMany({ where: { authorId: 1 } }),
]);
```

---

## 11. Daily Workflow Summary

Whenever you change your data model:

1. Edit `prisma/schema.prisma`.
2. Run `npx prisma migrate dev --name your_change_description`.
3. Prisma Client types update automatically — use the new fields in your code.
4. Commit both the schema change **and** the generated migration folder to git.

---

## 12. Deployment Checklist (Vercel / Serverless)

- [ ] Use a driver adapter or a connection pooler (e.g. PgBouncer, Prisma Accelerate, Neon's built-in pooling) — serverless functions open many short-lived connections and can exhaust a direct Postgres connection limit fast.
- [ ] Run `npx prisma migrate deploy` in your CI/CD pipeline, not `migrate dev`.
- [ ] Make sure `DATABASE_URL` is set as an environment variable in your hosting provider, not just locally.
- [ ] Run `npx prisma generate` as part of your build step (add it to `postinstall` in `package.json` if needed):
  ```json
  "scripts": {
    "postinstall": "prisma generate"
  }
  ```

---
## Remove every thing 

`npx prisma migrate reset`

---

## 13. Useful Commands Reference

| Command | Purpose |
|---|---|
| `npx prisma init` | Scaffold `schema.prisma` and `.env` |
| `npx prisma migrate dev --name x` | Create + apply a migration (dev) |
| `npx prisma migrate deploy` | Apply pending migrations (production) |
| `npx prisma generate` | Regenerate Prisma Client from schema |
| `npx prisma db pull` | Introspect an existing DB into the schema |
| `npx prisma db push` | Push schema changes without a migration file (prototyping only) |
| `npx prisma studio` | Open the visual data browser |
| `npx prisma format` | Auto-format the schema file |