# replit_kev — API Server

A TypeScript/Express API server scaffolded on Replit, meant to serve data from
the **CISA Known Exploited Vulnerabilities (KEV) catalog**. The server is
bundled into a single ESM file with esbuild and logs structured JSON with pino.

> **Status: early scaffold.** Only the health-check route is wired up. The KEV
> catalog endpoint is declared in the API schema (`GetKevCatalogResponse`) but
> has no route handler yet. See [Current state of the code](#current-state-of-the-code).

---

## Tech stack

| Concern        | Choice                                                   |
| -------------- | -------------------------------------------------------- |
| Runtime        | Node.js (ESM)                                            |
| Language       | TypeScript                                               |
| HTTP framework | Express                                                  |
| Logging        | pino + pino-http (pino-pretty transport)                 |
| Validation     | zod (response schemas)                                   |
| Build          | esbuild, with `esbuild-plugin-pino`                      |
| Middleware     | `cors`, `express.json()`, `express.urlencoded()`         |

## Project layout

This follows the Replit monorepo convention, where each deployable lives under
`artifacts/`:

```
replit_kev/
└── artifacts/
    └── api-server/
        ├── build.mjs              # esbuild bundler script → dist/index.mjs
        └── src/
            ├── index.ts           # entry point (starts the server)
            ├── app.ts             # Express app + middleware
            ├── lib/
            │   └── logger.ts      # shared pino logger
            └── routes/
                ├── index.ts       # root router, mounts sub-routers
                └── health.ts      # health-check route
```

Every route is mounted under the **`/api`** prefix.

## How it works

### App (`src/app.ts`)

The middleware runs in this order:

1. **`pino-http` request logging.** Custom serializers keep the logs small:
   requests log only `id`, `method` and `url` (the query string is stripped, so
   query parameters never end up in the logs), and responses log only
   `statusCode`. Headers, cookies and bodies are not logged.
2. **CORS** with default settings. This allows **all origins**, so tighten it
   before production if the API should only be reached from known frontends.
3. **Body parsing** for JSON and URL-encoded forms.
4. **The router**, mounted at `/api`.

### Routes (`src/routes/index.ts`)

The root router mounts sub-routers. Right now there is only one:

| Method | Path          | Description  | Response schema       |
| ------ | ------------- | ------------ | --------------------- |
| GET    | `/api/health` | Health check | `HealthCheckResponse` |

Planned:

| Method | Path       | Description                | Response schema         |
| ------ | ---------- | -------------------------- | ----------------------- |
| GET    | *(TBD)*    | Return the KEV catalog     | `GetKevCatalogResponse` |

### Build (`build.mjs`)

`build.mjs` bundles `src/index.ts` into `dist/index.mjs`:

- It deletes `dist/` first, so every build starts clean.
- Platform is `node` and the output format is ESM, with `.mjs` extensions and
  linked source maps.
- A **banner** adds `require`, `__filename` and `__dirname` to the global
  scope, so CommonJS-only dependencies such as Express keep working inside the
  ESM bundle.
- **`esbuild-plugin-pino`** handles pino's worker-thread transports
  (`pino-pretty`). Those can't be bundled the normal way.
- A long **`external` list** leaves packages that can't be bundled out of the
  bundle. These include native addons (`*.node`, `sharp`, `bcrypt`,
  `better-sqlite3`), packages that read sibling files at runtime
  (`@google-cloud/*`, `@grpc/*`) and large SDKs (`@aws-sdk/*`, `@azure/*`,
  `@prisma/client`). Many of them aren't installed. They are listed
  in advance so that adding one later doesn't break the build. Any package you
  add to this list must be installed in `node_modules` at runtime.

## Getting started

These commands assume a standard Replit pnpm workspace. Change the names if
your `package.json` scripts are different.

```bash
# install dependencies (from the repo root)
pnpm install

# build the API server
cd artifacts/api-server
node build.mjs

# run the bundle
node --enable-source-maps dist/index.mjs
```

Check that it's running:

```bash
curl http://localhost:$PORT/api/health
```

### Configuration

| Variable | Purpose                                                       |
| -------- | ------------------------------------------------------------- |
| `PORT`   | Port the server listens on. Replit sets this automatically.   |

## Current state of the code

The folder that was provided has **a single file**,
`artifacts/api-server/src/routes/index.ts`. Several files were pasted into it
one after another:

| Lines   | Actual file it came from          |
| ------- | --------------------------------- |
| 1–8     | `src/routes/index.ts`             |
| 9–42    | `src/app.ts`                      |
| 43–168  | `build.mjs`                       |
| 169–170 | generated zod schemas (cut off)   |

As a result, the project **won't compile as it stands**: there are duplicate
`default` exports and duplicate imports, and the zod object definitions are
incomplete. To get it working:

1. Split the file back into the files listed above.
2. Restore the missing files: `src/index.ts`, `src/lib/logger.ts`,
   `src/routes/health.ts`, `package.json` and `tsconfig.json`.
3. Restore the zod schemas file (`HealthCheckResponse`,
   `GetKevCatalogResponse`). In Replit templates this is usually generated
   from an OpenAPI spec in a shared `lib/` package.

## Roadmap

- [ ] Add a KEV route (for example `GET /api/kev`) that fetches the CISA KEV
      JSON feed, caches it, and validates it against `GetKevCatalogResponse`.
- [ ] Add filtering, for example by `cveID`, `vendorProject`, `dateAdded` or
      `knownRansomwareCampaignUse`.
- [ ] Restrict CORS to known origins.
- [ ] Add tests and a CI build check.
