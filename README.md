# Portfolio — Next.js + Sanity

A freelance portfolio site with its content in Sanity. Two workspaces:

```
studio/     Sanity Studio — schemas, structure, Presentation config
frontend/   The Next.js 16 site
```

Originally a file-based site (MDX + hardcoded constants), migrated so that every
page, every case study and all SEO are editable in the Studio.

## Running it

```bash
npm install
npm run dev          # Studio on :3333, site on :3000
```

Both workspaces need env files — copy the examples and fill them in:

```bash
cp studio/.env.example studio/.env
cp frontend/.env.example frontend/.env.local
```

| Script | What it does |
| --- | --- |
| `npm run dev` | Runs both workspaces via Turborepo |
| `npm run dev:next` / `npm run dev:studio` | One workspace only |
| `npm run lint` | ESLint over the frontend |
| `npm run type-check` | TypeScript across both workspaces |
| `npm run import-content` | One-off migration from the original file-based project |

## How content flows

```
Studio schema  →  sanity.schema.json  →  frontend/sanity.types.ts
    (studio/src/schemaTypes)              (generated, do not edit)

GROQ            frontend/sanity/lib/queries.ts
  ↓             projections normalise Sanity's shapes into plain objects
Validation      frontend/lib/content/schema.ts   (zod)
  ↓
Data layer      frontend/lib/content/sanity.ts   ← the only module that fetches
  ↓
Routes          frontend/app/**
```

Three things are worth knowing before changing any of it:

**Routes never fetch directly.** They import from `lib/content/sanity.ts`. If
content ever moves again, that one module is what gets rewritten.

**The zod schemas are not redundant.** The Studio enforces required fields when
an editor types; zod catches the other way the shape breaks — a projection that
stopped returning a field. It fails loudly, with the field path, instead of
rendering `undefined` into a page.

**Adding a field is a four-step change:** the Studio schema, the GROQ
projection, the zod schema, then the component. Miss the projection and zod
tells you at once.

### Regenerating types

`sanity.types.ts` is generated from the schema and the queries. It regenerates
automatically on `dev` and `build`, or manually:

```bash
npm run sanity:typegen --workspace=frontend
```

## Content model

| Type | Notes |
| --- | --- |
| `caseStudy` | The proof layer. 1–4 metrics required; `placeholder` flags invented details and renders a visible "sample" badge |
| `service` | The richest type. References other services as related; case studies, posts and testimonials all reference *it* |
| `post` | Journal entries |
| `testimonial` | References a service, optionally a case study |
| `page` | The keyed prose/index pages: work, blog, testimonials, privacy |
| Singletons | `settings`, `homePage`, `aboutPage`, `contactPage`, `servicesPage` |

Rich text is `portableText`, which carries the custom blocks the old MDX bodies
used: callout, metricRow, beforeAfter, figure, stackList, table and code. Each
maps to a component in `frontend/components/mdx/MdxComponents.tsx` via
`PortableBody.tsx`.

Service slugs used to be a hardcoded enum. They're documents now, and everything
that tags a service holds a *reference*, so a renamed service can't leave a
dangling tag behind.

## What isn't in the CMS, and why

- **`NEXT_PUBLIC_SITE_URL`** — differs per deployment. A CMS field would make
  preview builds emit production canonical URLs.
- **`NEXT_PUBLIC_CONTACT_EMAIL`** — the error and 404 pages can't fetch
  settings; the error boundary exists for when fetching is what broke.
- **Icons and diagrams** — `service.icon` and `service.diagram` are keys into
  code (`ServiceIcon.tsx`, `diagrams/Diagrams.tsx`). The Studio offers the
  known keys as a dropdown; adding a new one needs a code change, which is the
  trade for not bundling every Lucide icon.

## Visual editing

Set `SANITY_STUDIO_PREVIEW_URL` (studio) and `NEXT_PUBLIC_SANITY_STUDIO_URL`
(frontend), then open Presentation in the Studio. Draft Mode is enabled through
`frontend/app/api/draft-mode/enable`. Document-to-route resolution lives in
`studio/sanity.config.ts`.

## The content import

`studio/scripts/import-content.ts` migrated the original MDX and JSON into
Sanity. It's idempotent — deterministic document ids, written with
`createOrReplace` — so re-running updates rather than duplicating.

```bash
SOURCE_DIR="path/to/original/project" npm run import-content
npm run import-content -- --dry-run   # convert and report, write nothing
```

It needs an Editor token in `SANITY_API_WRITE_TOKEN`; the frontend's read token
isn't sufficient. The page copy it writes lives in `scripts/page-content.ts`.

The script has done its job — the Studio is the source of truth now. It's kept
because the conversion rules in `scripts/lib/markdown-to-portable-text.ts` are
the record of how the MDX mapped onto Portable Text.

## Deploying

See `vercel-installation-instructions.md` for the two-project Vercel setup.
A write token. Your SANITY_API_READ_TOKEN is read-only, so for the import I used your Sanity CLI's admin credentials, passed through the environment for that one command and not written to disk. For future runs, create an Editor token at manage.sanity.io → API → Tokens and put it in studio/.env as SANITY_API_WRITE_TOKEN.

Presentation click-test. npm run dev gives you Studio on :3333 and the site on :3000; preview URLs are wired. Confirming live visual editing needs a browser session, which I couldn't drive headlessly.

Before deploying, set NEXT_PUBLIC_SITE_URL (canonical URLs and OG links currently point at localhost) and the three RESEND_*/CONTACT_* vars for the contact form — all documented in the updated .env.example files and the rewritten README.




