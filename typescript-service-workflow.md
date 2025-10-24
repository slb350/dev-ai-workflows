# TypeScript Service Workflow - Node-Friendly, DB-Aware

**Version:** 1.0  
**Last Updated:** 2025-10-17  
**Purpose:** A backend-focused TypeScript workflow (Node 20+) that meshes with your SQL-first database process and multi-language environment.

---

## Overview

This playbook targets HTTP or worker services built with TypeScript/Node. It emphasizes:

- Modern ESM configuration.  
- `pnpm` or `npm` (with lockfile) for deterministic installs.  
- Unified lint/format via `eslint` + `prettier`.  
- Testing with `vitest` or `jest` (choose one).  
- Integration with shared PostgreSQL/SQLite artifacts.  
- Justfile automation to mirror Python/Go/Rust workflows.  
- Lightweight containers and deployment.

---

## Core Principles

1. **Strict Type Checking** - `tsconfig` locked to strictest options.  
2. **Zero Drift Formatting** - `prettier` + `eslint` enforce style.  
3. **Test First** - `vitest` or `jest` with coverage.  
4. **Database Owned by SQL Workflows** - Service consumes produced schema, no ORM auto-migrate.  
5. **Task Automation** - `just` to line up commands across stacks.  
6. **Modular Scripts** - Scripts in `package.json` map to `just` targets.  
7. **Documented Environment** - `.env.example`, docs, ADRs updated per feature.

---

## Tooling Baseline

| Category | Tool | Notes |
| --- | --- | --- |
| Runtime | Node.js 20 LTS | Use `fnm`, `asdf`, or `nvm` to pin version. |
| Package manager | `pnpm` (recommended) or `npm` with `--save-exact` | `pnpm` wastes less disk, but choose whichever fits project. |
| Build | `tsc --incremental`, optionally `tsx` for dev | `swc` or `esbuild` for fast bundling. |
| Testing | `vitest` (modern) or `jest` | Both support ESM; pick one to avoid duplication. |
| Lint/Format | `eslint` + `@typescript-eslint/*`, `prettier` | Use `eslint-config-prettier`. |
| Type checking | `tsc --noEmit`, optional `tsc --project tsconfig.build.json` for builds | Use `tsc --watch` in dev. |
| Observability | `pino`, `winston`, `opentelemetry` | Mirror Observability workflow. |
| Database integration | Shell/just commands that call Sqitch, pgTAP, tapsqlite | Keep schema external. |

Install base tooling (assuming `pnpm`):

```bash
corepack enable
corepack prepare pnpm@9 --activate
pnpm dlx create-ts-app my-service
```

Or start manually:

```bash
mkdir ts-service && cd ts-service
pnpm init
pnpm install typescript@5.4 @types/node@20 --save-dev
pnpm exec tsc --init --strict --module esnext --moduleResolution node16 --target es2022 --outDir dist --rootDir src --sourceMap true --resolveJsonModule true --noEmitOnError
```

---

## Project Structure

```text
ts-service/
├── src/
│   ├── index.ts
│   ├── env.ts
│   ├── routes/
│   ├── services/
│   └── db/ (adapters calling shared SQL)
├── tests/
│   ├── unit/
│   └── integration/
├── scripts/
│   ├── db-test.sh
│   └── lint-staged.mjs
├── justfile
├── package.json
├── pnpm-lock.yaml
├── tsconfig.json
├── tsconfig.build.json
├── .env.example
└── docs/
```

---

## Automation (`justfile`)

```just
set shell := ["bash", "-c"]

dotenv := "set -a && source .env && set +a"

alias ci := check

install:
	pnpm install

fmt:
	pnpm exec prettier --write "src/**/*.{ts,tsx}" "tests/**/*.ts"

lint:
	pnpm exec eslint "src/**/*.{ts,tsx}" "tests/**/*.ts"

typecheck:
	pnpm exec tsc --noEmit

test:
	pnpm exec vitest run --coverage

test-watch:
	pnpm exec vitest watch

build:
	pnpm exec tsc --project tsconfig.build.json

check:
	just fmt
	just lint
	just typecheck
	just test

db-test:
	./scripts/db-test.sh

integration:
	just db-test
	pnpm exec vitest run tests/integration --coverage=false

dev:
	pnpm exec tsx watch src/index.ts
```

Replace `vitest` with `jest` if preferred; adjust scripts accordingly.

---

## `package.json` Scripts (example)

```json
{
  "type": "module",
  "scripts": {
    "build": "tsc --project tsconfig.build.json",
    "start": "node dist/index.js",
    "dev": "tsx watch src/index.ts",
    "test": "vitest run",
    "test:watch": "vitest watch",
    "lint": "eslint \"src/**/*.{ts,tsx}\" \"tests/**/*.ts\"",
    "lint:fix": "eslint --fix \"src/**/*.{ts,tsx}\" \"tests/**/*.ts\"",
    "format": "prettier --write \"src/**/*.{ts,tsx}\" \"tests/**/*.ts\"",
    "type-check": "tsc --noEmit",
    "db:test": "./scripts/db-test.sh",
    "ci": "pnpm lint && pnpm type-check && pnpm test"
  },
  "lint-staged": {
    "src/**/*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ]
  }
}
```

Create `tsconfig.build.json` for production builds (exclude test files).

---

## Environment Configuration

```text
# .env.example
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/app_dev
SQLITE_DB_PATH=../databases/app.sqlite
LOG_LEVEL=info
```

Use `dotenv-flow` or `tsx` to load env files, but never commit the real `.env`.

---

## TDD Cycle

### Step 1: RED

```ts
// tests/unit/user-service.test.ts
import { describe, expect, it } from 'vitest';
import { createUser } from '../../src/services/user-service';

describe('createUser', () => {
  it('creates a user when input valid', async () => {
    const result = await createUser({ email: 'demo@example.com', name: 'Demo' });
    expect(result.isOk()).toBe(true);
    expect(result.unwrap().email).toBe('demo@example.com');
  });
});
```

Run `just test` (fails).

### Step 2: GREEN

```ts
// src/services/user-service.ts
import { Result, err, ok } from '@sniptt/monads';
import { z } from 'zod';

const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1),
});

type CreateUserInput = z.infer<typeof CreateUserSchema>;

export async function createUser(payload: CreateUserInput): Promise<Result<CreateUserInput, string>> {
  const parse = CreateUserSchema.safeParse(payload);
  if (!parse.success) {
    return err('invalid payload');
  }
  // TODO: persist via repository
  return ok(parse.data);
}
```

Tests pass? Move on.

### Step 3: REFACTOR

- Inject repository interface to persist to Postgres/SQLite.  
- Add logging with `pino`.  
- Rerun `just fmt` and `just lint`.  
- Update docs.

### Step 4: COMMIT

```bash
git status
git add src tests package.json justfile tsconfig*.json .env.example docs/
git commit -m "feat: add user service scaffold"
```

---

## Database Integration

1. Run shared SQL workflows (PostgreSQL/SQLite) to prepare schema and tests.  
2. `just db-test` executes `scripts/db-test.sh`, which might call:
   - `sqitch deploy --verify`  
   - `pg_prove` or `sqlite3` TAP tests  
   - Data fixtures import  
3. Integration tests connect to the prepared DB:

```ts
import { beforeAll, afterAll } from 'vitest';
import { Pool } from 'pg';

let pool: Pool;

beforeAll(async () => {
  pool = new Pool({ connectionString: process.env.DATABASE_URL });
  await pool.query('SELECT 1'); // ensure ready
});

afterAll(async () => {
  await pool.end();
});
```

1. For SQLite, copy the pristine file into temp directories before each test.

---

## Packaging & Deployment

- **Build**: `pnpm run build` creates `dist`.  
- **Run**: `node dist/index.js` (with env loaded).  
- **Dockerfile**:

```Dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
RUN pnpm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/package.json /app/pnpm-lock.yaml ./
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
ENV NODE_ENV=production
CMD ["node", "dist/index.js"]
```

- For serverless (Cloudflare Workers, AWS Lambda), consider bundling with `esbuild` or `tsup`.

---

## Checklists

**Daily Commit**  

- [ ] `just check` (fmt, lint, typecheck, tests).  
- [ ] `just db-test` run if schema touched.  
- [ ] README/docs updated.  
- [ ] `.env.example` reflects new config.  
- [ ] `git status` clean.

**Release**  

- [ ] `pnpm run build` succeeded.  
- [ ] Docker image built/pushed (if applicable).  
- [ ] Migrations bundle exported (`sqitch bundle`).  
- [ ] CHANGELOG entry added.  
- [ ] Observability config validated (logs/traces).  
- [ ] Smoke tests defined for deployment target.

---

## Anti-Patterns

- Using `ts-node` in production (bundle/compile first).  
- Allowing ORMs (Prisma/TypeORM/Drizzle) to auto-migrate without SQL review.  
- Relying on CommonJS when the ecosystem expects ESM.  
- Skipping type checking because tests pass.  
- Missing `pnpm-lock.yaml` in version control.  
- Hardcoding credentials instead of `.env`.

---

## Success Metrics

- ✅ `just check` completes quickly (<90 seconds).  
- ✅ Integration tests use shared database artifacts.  
- ✅ Build artifacts reproducible via `pnpm run build`.  
- ✅ Observability instrumentation toggled via env.  
- ✅ Documentation always explains bootstrap/run/test steps.

---

## Troubleshooting

| Problem | Fix |
| --- | --- |
| ESM import errors | Ensure `"type": "module"`, use `.js` extension for compiled output, configure Jest/Vitest for ESM. |
| Lint conflicts | Add `eslint-config-prettier`, tune `.eslintrc`. |
| Type errors from DB libs | Install appropriate `@types/*` or rely on generated types from SQL queries. |
| Tests hitting dead DB | Run `just db-test`, ensure DSN points to local container/test sqlite file. |
| Build size too big | Bundle with `esbuild --minify` or use tree-shaking. |
| Slow install | Use `pnpm fetch` + `pnpm install --offline` in CI. |

---

**Remember:** TypeScript services remain lightweight when the database contract lives outside the app. Keep the service a thin layer around typed business logic and rely on your SQL workflows for schema correctness. 📘
