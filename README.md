# Container Storage Provider (CSP)

The Container Storage Provider (CSP) is a REST API specification that defines how provisioning, mounting, and deallocation workflows will be invoked from a host client that intends to use a storage provider.  Any storage vendor that intends to use the [HPE CSI Driver](https://github.com/hpe-storage/csi-driver) must implement the specification defined herein.

This repo contains the CSP [specification](spec.md) file.

You may inspect a Swagger rendering of the API specification on [Redocly](https://hpe-csi-driver-csp-api.redoc.ly/).

## OpenAPI contract (source of truth)

The machine-readable API contract is authored here as OpenAPI 3.x. The authored
source of truth is [`openapi/openapi.yaml`](openapi/openapi.yaml); the rendered
`nimble-storage/nimble-csp-api-openapi.{yaml,json}` files are build outputs
produced by `npm run bundle`. See [`openapi/README.md`](openapi/README.md) for
the contract-first workflow — edit, lint, bundle, and how to generate a Redocly
preview (`npm run preview` for a live server, `npm run preview:build` for a
shareable static HTML) to review changes before merging.

