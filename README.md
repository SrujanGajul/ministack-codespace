# MiniStack on GitHub Codespaces

Lightweight devcontainer that runs the [MiniStack](https://ministack.org) AWS
emulator plus the [StackPort](https://github.com/DaviReisVieira/stackport) web
UI inside a free-tier GitHub Codespace. Tunnel ports `4566` (AWS API) and
`8080` (web UI) back to your local laptop.

Parallel to the [floci-codespace](https://github.com/SrujanGajul/floci-codespace)
repo — same architecture, MiniStack instead of Floci. MiniStack is the default
emulator that StackPort itself recommends, so this combo is the most-tested path.

## Why MiniStack vs Floci

| | MiniStack | Floci |
|---|---|---|
| Language | Python | Java (Quarkus native) |
| Image size | ~270 MB | ~80 MB |
| Idle RAM | ~21 MiB | ~13 MiB |
| Startup | ~2 s | ~24 ms |
| Releases | 130+, 70 contributors | newer, smaller team |
| Real infra (RDS/ECS/Lambda) | yes (via docker.sock) | yes (via docker.sock) |
| StackPort first-class support | **yes (default)** | yes (LocalStack wire-compat) |
| License | MIT | MIT |
| Auth required | no | no |

Pick MiniStack when you want maturity + broader feature parity. Pick Floci when
you want minimum resource footprint.

---

## Architecture

```text
+---------------------+         +---------------------------------------+
|  Your local laptop  |         |        GitHub Codespace VM            |
|                     |  WSS    |                                       |
|  aws cli  :4566     |<-tunnel-+--+  Dev container (ubuntu-base)        |
|  browser  :8080     |<-tunnel-+  +--+ docker-in-docker                 |
|                     |         |     +--+ ministack  :4566 (AWS API)   |
|                     |         |     +--+ stackport  :8080 (Web UI)    |
+---------------------+         |        +--+ ministack-data volume     |
                                +---------------------------------------+
```

The `docker.sock` is bind-mounted into MiniStack so it can spawn sibling
containers for RDS (Postgres), ElastiCache (Redis), ECS, and Lambda.

---

## One-time setup

```bash
# clone (skip if you already pushed it)
git clone git@github.com:SrujanGajul/ministack-codespace.git
cd ministack-codespace

# gh CLI auth (skip GITHUB_TOKEN env var if it shadows your personal account)
unset GITHUB_TOKEN
gh auth status
```

---

## Quick start: create -> tunnel -> use

```bash
unset GITHUB_TOKEN

# 1. Create
gh codespace create \
  -R SrujanGajul/ministack-codespace \
  -m basicLinux32gb \
  -b main \
  --display-name ministack

# 2. Wait for state = Available
gh codespace list

# 3. Forward ports to your local laptop (run in two terminals to avoid the
#    occasional 400 race when forwarding both in one command)
gh codespace ports forward 4566:4566 -c ministack-<id> &
gh codespace ports forward 8080:8080 -c ministack-<id> &

# 4. Open web UI
xdg-open http://localhost:8080

# 5. Use the AWS CLI from a fresh terminal
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test

aws s3 mb s3://test
echo hi | aws s3 cp - s3://test/hello.txt
aws s3 ls s3://test/
# Refresh StackPort tab to see the bucket and object.
```

---

## Lifecycle commands

| Action | Command |
|---|---|
| Create | `gh codespace create -R SrujanGajul/ministack-codespace -m basicLinux32gb -b main --display-name ministack` |
| List | `gh codespace list` |
| Open VS Code | `gh codespace code -c <name>` |
| SSH in | `gh codespace ssh -c <name>` |
| Forward port | `gh codespace ports forward 4566:4566 -c <name>` |
| List ports | `gh codespace ports -c <name>` |
| Make port public | `gh codespace ports visibility 4566:public -c <name>` |
| Stop (pause) | `gh codespace stop -c <name>` |
| Rebuild | `gh codespace rebuild -c <name>` |
| Full rebuild | `gh codespace rebuild --full -c <name>` |
| Delete | `gh codespace delete -c <name>` |

---

## Pause / resume

Codespaces auto-stop after 30 min idle (default, max 240). When stopped:

- Container state preserved (volumes, files)
- Compute quota stops
- Tunnel drops (`websocket: close 1006`) — expected

Resume with any of `gh codespace ports forward / ssh / code` — they auto-wake.
MiniStack and StackPort both have `restart: unless-stopped`, so they auto-start
when the codespace resumes.

---

## Smoke test (full)

```bash
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test

# S3
aws s3 mb s3://e2e
echo "hello from local" | aws s3 cp - s3://e2e/hello.txt
aws s3 ls s3://e2e/

# SQS
aws sqs create-queue --queue-name orders
aws sqs send-message --queue-url http://localhost:4566/000000000000/orders --message-body '{"x":1}'

# DynamoDB
aws dynamodb create-table --table-name Users \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
aws dynamodb put-item --table-name Users --item '{"id":{"S":"u1"},"name":{"S":"Alice"}}'

# Lambda
aws lambda list-functions

# RDS (this actually spawns a real Postgres container via docker.sock)
aws rds describe-db-instances
```

---

## StackPort web UI

| URL | Notes |
|---|---|
| `http://localhost:8080` (via tunnel) | Recommended |
| `https://<codespace>-8080.app.github.dev` | Direct Codespaces HTTPS URL |

Features: dashboard, dedicated browsers for S3, DynamoDB, Lambda, SQS, IAM,
EC2, CloudWatch Logs, Secrets Manager; generic resource table for the rest;
tag management; live updates via WebSocket. Write ops enabled
(`STACKPORT_ALLOW_WRITES=true`).

Note: StackPort is an **explorer**, not a full AWS Console replica. Resource
creation (e.g. `aws s3 mb`, `create-table`) typically happens via the AWS CLI;
StackPort then lets you inspect and manipulate the data inside.

---

## Files in this repo

| File | Purpose |
|---|---|
| [.devcontainer/devcontainer.json](.devcontainer/devcontainer.json) | Devcontainer config: features (DinD, aws-cli, sshd), env vars, port forward, postCreate |
| [.devcontainer/docker-compose.yml](.devcontainer/docker-compose.yml) | MiniStack service (`:4566`) + StackPort web UI (`:8080`), named volume, docker.sock bind-mount |
| [README.md](README.md) | This document |
| [.gitignore](.gitignore) | Ignore local data dumps, .env |

---

## Troubleshooting

### Tunnel `websocket: close 1006` — Codespace went idle. Resume with any gh codespace command.

### Recovery container — Codespaces prebuild glitch. `gh codespace delete -c <id> --force` and recreate.

### `postCreateCommand` skipped on rebuild — known race; SSH in and run `cd /workspaces/ministack-codespace && docker compose -f .devcontainer/docker-compose.yml up -d`.

### `Permission denied (publickey)` on push — SSH key belongs to another GH account. Switch via `gh auth switch -u <user>` before `gh repo create`, or change remote to HTTPS.

### `GITHUB_TOKEN` shadows interactive auth — `unset GITHUB_TOKEN` for any codespace operation on your personal repo.

### MiniStack RDS / ECS not working — confirm `/var/run/docker.sock` is bind-mounted into the container.
