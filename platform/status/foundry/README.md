# Foundry status files

This folder is updated automatically by the workflow:

- .github/workflows/foundry-provision.yml

A status file is generated per request as:

- platform/status/foundry/<request-name>.yaml

The status includes:

- request reference
- run metadata
- current state (`success` or `failure`)
- selected deployment outputs when available

Do not edit status files manually.
