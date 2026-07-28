# Contributing to jackpoll-selfhost

This repo holds the self-hosting deployment: Docker Compose / Swarm stacks,
env templates, and the Keycloak realm. No application source code lives
here — that's in [jackpoll](https://github.com/jackpoll-org/jackpoll).

## What belongs here

- Fixes/improvements to `docker-compose.simple.yml` / `docker-compose.swarm.yml`.
- Documentation fixes in [SELF_HOSTING.md](SELF_HOSTING.md) / [README.md](README.md).
- New optional services or deployment paths, if discussed in an issue first.

Application bugs (survey builder, API behavior, mobile app) belong in the
[jackpoll](https://github.com/jackpoll-org/jackpoll) repo instead.

## Before opening a PR

Validate both compose files parse and resolve correctly:

```bash
cp .env.simple.example /tmp/test.env
docker compose --env-file /tmp/test.env -f docker-compose.simple.yml config -q
```

Swarm changes are harder to test locally (no `docker compose config` equivalent
for stack files) — at minimum lint the YAML and check the Traefik label
syntax carefully against the existing services.

If you change `keycloak/survey-school-realm.json`, note that it's synced
from the [jackpoll monorepo](https://github.com/jackpoll-org/jackpoll/tree/main/apps/backend/src/main/keycloak)
by `.github/workflows/sync-realm.yml` — changes should usually go there
instead, or they'll be overwritten on the next sync.

## Reporting issues

Open a GitHub issue with your deployment path (simple/Swarm), Docker version,
and the relevant logs (`docker compose logs <service>` or
`docker service logs survey_<service>`).

## Security issues

Do **not** open a public issue for a security vulnerability — see
[SECURITY.md](SECURITY.md).

## License

By contributing, you agree your contribution is licensed under this repo's
[MIT license](LICENSE).
