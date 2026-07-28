# Security Policy

## Reporting a vulnerability

**Do not open a public GitHub issue for security vulnerabilities** — this
repo covers self-hosted deployments, so a public report could point at
exposed instances before they can be patched.

Report privately instead:

- Preferred: [GitHub Security Advisories](https://github.com/jackpoll-org/jackpoll-selfhost/security/advisories/new)
- Or email [contact@jackpoll.org](mailto:contact@jackpoll.org)

Include what you can: which stack (simple/Swarm), what's misconfigured or
exploitable, and impact.

## Scope

This repo's deployment configuration (compose files, env templates, Keycloak
realm, Traefik labels). Vulnerabilities in the application itself (frontend/
backend code) should go to
[jackpoll's security policy](https://github.com/jackpoll-org/jackpoll/security/policy)
instead.

## A note on defaults

Every secret in `.env.simple.example` / `.env.swarm.example` is a
`CHANGE_ME`/`change-me` placeholder — the backend refuses to boot on `master`
if a required one is left unset. If you find a secret that's missing this
fail-fast behavior, that's a vulnerability report, not a docs nitpick.
