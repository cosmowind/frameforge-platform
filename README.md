# FrameForge Platform

[![CI](https://github.com/cosmowind/frameforge-platform/actions/workflows/ci.yml/badge.svg)](https://github.com/cosmowind/frameforge-platform/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

**A multi-tenant ingest-and-publish platform for camera footage.**

Footage arrives on a memory card. This turns it into a catalogued, tagged,
published asset without anyone renaming files by hand.

```
 memory card                 FrameForge Platform                 object storage
 ┌──────────────┐      ┌───────────────────────────┐      ┌──────────────────┐
 │ DCIM/ hmd/   │ ───▶ │ 1. pick device + scan     │ ───▶ │ S3 / COS         │
 │ *.mp4 *.hmd  │      │ 2. preview in the browser  │      │ signed upload    │
 │              │      │ 3. fill metadata, publish │      │ + lifecycle      │
 └──────────────┘      └───────────────────────────┘      └──────────────────┘
```

---

## What it is

A Next.js monorepo with three deployable units:

| App | Port | What it does |
|---|---|---|
| `apps/web-uploader` | 3001 | The operator-facing surface: pick a device, scan, preview, fill the metadata form, upload. |
| `apps/web-admin` | 3002 | The back office: dimensions, quotas, scene catalogue, audit trail, raw database view. |
| `apps/worker` | 3010 | BullMQ consumers: device scan, preview generation, pull from cold storage, scheduled backup. |

Plus two shared packages (`packages/shared`, `packages/db-schema`) and one
adapter package (`packages/adapters`).

## The three-step flow

```
[card plugged in]
      │  Step 1  pick device type + scan        (nothing is persisted yet --
      ▼                                         the file list lives in the browser)
[scanned]
      │  Step 2  preview                        (native <video>, an HMD decoder,
      ▼                                          an MCAP extractor, or an AVI concat)
[previewed]
      │  Step 3  fill the form + publish        (the adapter prepares the payload,
      ▼                                          the browser uploads with a signed URL)
[published]
```

A recording moves through `SCANNED → UPLOADING → UPLOADED → LOCAL_DELETED → ARCHIVED`.

## Device adapters

The interesting part of this codebase is `packages/adapters`. Every camera
family has different folder conventions, different preview strategies and
different upload payloads, and the rest of the platform does not care which
one it is talking to.

| code | device | card layout | preview | publish |
|---|---|---|---|---|
| `dji-gopro-dcim` | DJI / GoPro | `DCIM/` | native HTML5 | direct |
| `head-mounted-hmd` | head-mounted stereo | `hmd/` | hmd-decoder | extract-hmd |
| `gopro-session-mcap` | GoPro session | `data/` | mcap-extract | extract-mcap |
| `generic-avi-recording` | generic AVI recorders | `recordings/<date>_<ts>/` | native + JS playback order | concat-avi |

Writing a new adapter means implementing `DeviceAdapter` and registering it --
see [`docs/device-adapters.md`](docs/device-adapters.md).

## Stack

| layer | choice |
|---|---|
| web | Next.js 15, App Router, Turbopack |
| workers | Node, BullMQ on Redis 7 |
| data | PostgreSQL 16, Drizzle ORM, schema `ff`, table prefix `ff_*` (15 tables) |
| auth | Clerk (webhook-synced into `ff_users`) |
| storage | S3-compatible object storage, with a local-disk tier and a cold tier |
| local | Docker Compose: Postgres + Redis + MinIO + the three apps |

## Quick start

```bash
npm i -g pnpm@10.19.0

cp .env.example apps/web-uploader/.env
cp .env.example apps/web-admin/.env
cp .env.example apps/worker/.env

docker compose up -d          # Postgres + Redis + MinIO
pnpm install
pnpm db:migrate               # enums + tables
pnpm db:seed                  # default region + device types
pnpm dev                      # uploader :3001, admin :3002, worker :3010
```

Nothing here needs a real cloud account to start: point `S3_*` at the bundled
MinIO and it behaves like the real thing.

## Configuration

Everything is environment-driven; `.env.example` documents every variable.
The ones that matter most:

| variable | meaning |
|---|---|
| `DATABASE_URL` | Postgres connection string |
| `DATABASE_SCHEMA` | schema name, `ff` by default |
| `REDIS_URL` | Redis for BullMQ |
| `S3_ENDPOINT` / `S3_BUCKET` / `S3_ACCESS_KEY_ID` / `S3_SECRET_ACCESS_KEY` | object storage |
| `CLERK_SECRET_KEY` | auth (leave unset in development) |

## Repository layout

```
.
├── apps/
│   ├── web-uploader/        Next.js 15 (operator surface)
│   ├── web-admin/           Next.js 15 (back office)
│   └── worker/              Node + BullMQ consumers
├── packages/
│   ├── shared/              Zod schemas, shared types, the DeviceAdapter interface
│   ├── db-schema/           Drizzle definitions for the 15 ff_ tables
│   └── adapters/            one DeviceAdapter per camera family
├── infra/docker/            multi-stage Dockerfiles
├── deploy/                  deployment scripts and nginx config
├── docs/                    design notes, data dictionary, error book
└── docker-compose.yml       Postgres + Redis + MinIO + the three apps
```

## Documentation

| document | what it covers |
|---|---|
| [`docs/project_status.md`](docs/project_status.md) | current architecture and what is actually done |
| [`docs/structure.md`](docs/structure.md) | directory map and module responsibilities |
| [`docs/dev_plan.md`](docs/dev_plan.md) | roadmap |
| [`docs/dev_log.md`](docs/dev_log.md) | what was built, how it was verified, what went wrong |
| [`docs/error_book.md`](docs/error_book.md) | post-mortems of past failures |
| [`docs/data-dictionary.md`](docs/data-dictionary.md) | field-by-field reference for the 15 tables |
| [`docs/storage-architecture.md`](docs/storage-architecture.md) | the storage tiers |
| [`docs/device-adapters.md`](docs/device-adapters.md) | how to add a camera family |

A Chinese-language version of the original project notes is kept in
[`README.zh-CN.md`](README.zh-CN.md).

## Status

This is a published snapshot of an internal system. The schema, the admin
console and the adapters are solid; worker operations are thin, and the
documentation in `docs/dev_log.md` is honest about which parts are
production-hardened and which are not.

## License

MIT -- see [LICENSE](LICENSE).
