# PRODUCT.md

## Prefix
AEQ

## Domain
canon

## External API
GitHub raw content API — fetches canon.json from the pipeline authority repo at
https://raw.githubusercontent.com/tawni-dev/iterum-pipeline/main/canon.json
Node service fetches server-side on a configurable cache TTL, compares approved
SHAs against upstream release tag metadata via GitHub API, and returns the
normalized registry with staleness metadata. No write operations. No user data
in the request or response.

## Reference Authority
The public `tawni-dev/iterum-pipeline` repository is the Iterum Reference
Authority for learning, evaluation, demonstrations, and prototypes. This demo
reads and displays its public canon. It contains no credentials, write access,
private tokens, or administrative operations against the pipeline repository.
Publishing the demo and its source code does not expose other users' pipelines
by itself.

The demo is a reference implementation of the pipeline visibility model, not a
control plane for consumer repositories. Long-lived and production projects
should maintain their own pipeline authority and `canon.json`; changing the
configured canon URL replaces the authority the project consumes.

## Database
PostgreSQL — canon snapshots table. Stores each fetched canon state with a
timestamp. Read-only from the API. Provides a drift timeline: when values
changed, what changed, how long each version was active. Reference data only.
No user-identifying columns. No PII surface.

## Logging destination
Application Insights — structured traces only. No user input logged.
searchTerm UUID per error occurrence, never in public response.

## Interface
Three-tab layout. Registry tab: live canon values displayed as a formatted table
showing action names, approved SHAs (truncated with copy-on-click), tool versions,
base image digests, and backend dependency versions — each row shows staleness
metadata (last updated timestamp, how long this value has been active). Pipeline
tab: the 14-stage map rendered as a visual sequence, each stage expandable to show
its requirement description from pipeline-os.md. Docs tab: static content rendered
from README.md and setup guide — the two-phase flow diagram, phase instructions,
and links to the spec files. No forms. No user input anywhere in the application.
