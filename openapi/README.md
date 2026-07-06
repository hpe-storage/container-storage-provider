# CSP OpenAPI — contract-first source

This directory holds the **authored, OpenAPI 3.x source of truth** for the HPE
Container Storage Provider (CSP) API.

- `openapi/openapi.yaml` — the authored contract (edit this).
- `nimble-storage/nimble-csp-api-openapi.{yaml,json}` — **build outputs** produced
  by `npm run bundle`. Do not hand-edit them.
- `nimble-storage/nimble-csp-api-swagger-v2.{yaml,json}` — the **legacy Swagger 2.0**
  rendering. Kept frozen during migration until all consumers move to the OAS3
  files, then removed.

## Why this exists (contract-first bootstrap)

Historically the swagger file was generated **code-first** from Java/Swagger
annotations in the Nimble CSP implementation repo and copied here by hand. This
directory inverts that: the contract is authored **here**, in the public spec
repo, independent of any single vendor's implementation. See the CSP spec in
[`../spec.md`](../spec.md).

## One-time bootstrap

Seed the authored OAS3 source by converting the existing Swagger 2.0 rendering
(no dependency on the implementation repo, no Java/Maven):

```bash
npm install
npm run bootstrap      # swagger2openapi: nimble-csp-api-swagger-v2.yaml -> openapi/openapi.yaml
```

## Everyday workflow

```bash
npm install       # once, installs @redocly/cli + swagger2openapi
npm run lint      # validate the authored spec (redocly lint)
npm run bundle    # produce nimble-storage/nimble-csp-api-openapi.{yaml,json}
npm run build     # lint + bundle
```

Step by step, to make and land a change:

1. **Edit** `openapi/openapi.yaml` (the only file you author).
2. **Validate:** `npm run lint`.
3. **Regenerate outputs:** `npm run bundle` (or `npm run build` to lint + bundle
   in one shot).
4. **Preview** the rendered docs (see below) and eyeball the change.
5. **Commit both** the edited source and the regenerated
   `nimble-storage/*-openapi.*` outputs in the same commit. CI
   (`.github/workflows/openapi.yml`) lints the source and fails if the committed
   renderings are out of date.

## Previewing the docs (Redocly)

Two ways to render the spec for review before merging:

```bash
# Live, hot-reloading preview server (best while iterating locally)
npm run preview          # redocly preview-docs → http://localhost:8080

# Static, self-contained HTML (best to attach to a PR / share with reviewers)
npm run preview:build    # redocly build-docs → csp-api-preview.html
open csp-api-preview.html   # macOS (or open it in any browser)
```

- Both render the `nimble-csp-api` API defined in `redocly.yaml`, i.e. the
  authored `openapi/openapi.yaml`.
- `csp-api-preview.html` is a single self-contained file (git-ignored) that
  reviewers can open without any tooling installed.
- To preview the exact `$ref`-inlined artifact consumers receive, run
  `npm run bundle` first, then point `build-docs` at the bundle:
  `npx @redocly/cli build-docs nimble-storage/nimble-csp-api-openapi.yaml -o csp-api-preview.html`.
- See *what changed* at the API level (added/removed/breaking operations):
  `npx @redocly/cli diff <base-spec> nimble-storage/nimble-csp-api-openapi.yaml`.

## Optional: split for maintainability

For larger specs, split into multiple files and let Redocly bundle them:

```bash
npx @redocly/cli split openapi/openapi.yaml --outDir openapi/
```
