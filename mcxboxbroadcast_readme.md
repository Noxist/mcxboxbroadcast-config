# MCXboxBroadcast

## Purpose

Standalone Xbox Live session broadcaster for the exaroton SMP, so console players can see the server through the bot account's friends list.

## Architecture

The container runs on this Linux host and advertises the Bedrock endpoint hosted by exaroton. Minecraft traffic is expected to go directly through exaroton/Geyser; this host does not proxy gameplay or publish an inbound port. Static fallback metadata keeps the broadcast session independent of a failed server query. exaroton's current Geyser guidance specifies Bedrock port 19132; its NetherNet guidance says this standard port is used for all servers.

## Location

`/home/leandro/services/mcxboxbroadcast/`

## Runtime

Docker Compose using the official `ghcr.io/mcxboxbroadcast/standalone` image, pinned to Linux/amd64 digest `sha256:bd00c168e3183738dc364b0da9a6df83f1e809124ee0698f0b3fdd691054118a` (Build 162, commit `e7ef6b0`). `restart: unless-stopped` is enabled.

## Services and Containers

`mcxboxbroadcast` is the only project service. The container is running as UID/GID 1001, with supplementary GID 1000 for restricted access to the project config. Docker reports state `running`; the image has no Docker healthcheck. Dedicated-account authentication is pending.

## Networking

The container requires outbound internet access for Xbox authentication/session APIs and outbound connectivity to the exaroton Bedrock endpoint. It publishes no host ports and uses Docker's default bridge networking. This project adds no IPv4 or IPv6 inbound exposure.

## Public Access

The configured target is `ktw.nox007.tech:19132`, hosted by exaroton. Cloudflare delegates only `ktw.nox007.tech` to `columbus.exaroton.com` and `galileo.exaroton.com`; both public resolvers return exaroton A records, and the delegated zone returns Java SRV TCP 25565. MCXboxBroadcast's Bedrock status query to the custom host completed without a ping failure. Bedrock uses explicit UDP 19132, not Java SRV records. No player join has been tested. No service on this root server is exposed publicly.

## Configuration

- Compose: `docker-compose.yml`
- MCXboxBroadcast config and nonsecret runtime state: `config/`
- Current generated config is version 9, from official standalone Build 162. The target is `session.remote-address` / `remote-port` (`ktw.nox007.tech:19132`). It queries server status and uses static fallback data if a query fails, uses 60-second friend sync, accepts friend requests, invites newly added friends, and disables friend expiry. The current build has no separate `auto-follow` key.

## Secrets

| Name | Purpose | Storage location | Scope/Notes |
|---|---|---|---|
| Xbox account token cache | MCXboxBroadcast session authentication | `/etc/server-secrets/mcxboxbroadcast/cache.json` via a symlink at `config/cache/cache.json` | Personal-account cache for `[redacted-gamertag]` was removed during migration. New dedicated-account cache will be created after device login. Secret directory is mode 0700 and owned by container UID 1001. |

## Persistent Data

The Compose bind mount persists config, player history, rotated app logs, and session files under `config/`. The auth cache is symlinked to the dedicated secret directory above. The project `config/` directory is setgid mode 2770 for the `leandro` account and container GID 1000; the generated config is mode 0660. The auth cache is currently absent until Microsoft authentication completes.

## Backup

Before a config change, copy `config/` to a dated directory under `backups/`; preserve the auth-cache symlink and do not copy or expose its target casually. `backups/pre-build162-migration-20261109/` contains the prior config and Compose file; `backups/pre-custom-domain-probe-20261109/` contains the pre-switch config. The personal token cache was intentionally not backed up; other cache data was preserved.

## Restore

Stop the service, restore the intended config backup into `config/` while preserving restrictive ownership and permissions, then start the Compose service. Do not restore a token cache from an untrusted backup.

## Deployment / Start / Stop

From this directory:

- Start: `docker compose up -d`
- Stop: `docker compose down`
- Restart: `docker compose restart`

These commands affect only this project's container. Docker's `unless-stopped` policy restarts it after daemon/host restart unless explicitly stopped.

## Update Procedure

Review the upstream release and schema, update the image digest in Compose, pull the new image, back up the config, and recreate only this container with `docker compose up -d`.

## Health Checks

Check `docker compose ps` and inspect startup/session logs. The container is running, but has no embedded healthcheck. Dedicated-account device login is pending, so no current Xbox session is established. End-to-end visibility/join testing requires a real console.

## Logs

Use `docker compose logs --tail=100` from this directory. Docker JSON logs are rotated at 10 MB with three files. Never share authentication/device-code output publicly.

## Security Model

The broadcaster needs outbound Xbox connectivity and stores an Xbox token after interactive device authentication. The bot must use a dedicated account with an Xbox profile. No host ports, Docker socket, host networking, or unrelated service changes are part of this project. The process is non-root. Its secret cache uses a separate mode-0700 directory, bind-mounted only to the app's expected secret path; other persistent data stays under the project directory.

## Security Exceptions

The app requires its token cache at a fixed relative path (`./cache/cache.json`). A symlink redirects only that cache file into `/etc/server-secrets/mcxboxbroadcast/`, which is bind-mounted as `/run/secrets/mcxboxbroadcast`; the app's other cache data remains under the requested project directory.

## Rollback

Rollback is `docker compose down`; this stops/removes the container and project network but preserves bind-mounted data. Preserve `config/`, `backups/`, and `/etc/server-secrets/mcxboxbroadcast/`; do not remove credentials during routine rollback.

## Troubleshooting

- Offline query: static fallback activates if the Bedrock status query fails. The current Build 162 query to `ktw.nox007.tech:19132` did not log a failure or fallback; this still does not prove a player join works.
- No session shown: verify authentication, bot gamertag friendship, the Bedrock host/UDP port, and session logs.
- Offline join: exaroton supports automatic startup for Java and Bedrock and an offline waiting lobby. Enable both in the panel. Verify with a real console; static session metadata alone does not prove the console's actual join reaches the exaroton lobby.
- DNS: Cloudflare child NS delegation is active and public resolvers return exaroton A/SRV data. The `HTTPS → :80` panel route is for exaroton web services, not Minecraft; it is separate and should not be changed to a game port. The exaroton panel's current Pending label was not independently rechecked; DNS changes may take up to 24 hours to show there.

## Maintenance Notes

- Host is Ubuntu 22.04.5 LTS, amd64; Docker Engine 29.9.0 and Compose v5.6.0 are present.
- Java is not installed; the existing Docker runtime is preferred.
- Filesystem space was freed and the pinned image pull completed; no unrelated images or volumes were pruned.
- Cloudflare zone `nox007.tech` was verified through its API. Two records were created only at `ktw.nox007.tech`: NS to `columbus.exaroton.com` and NS to `galileo.exaroton.com`, both DNS-only with automatic TTL. Public resolvers 1.1.1.1 and 8.8.8.8 return the delegation.
- The original `ktws.exaroton.me` resolves to `37.228.148.22` and `37.228.148.23`; its Java SRV record points to TCP 25565. The delegated custom zone returns the same A targets and an SRV to TCP 25565. Bedrock uses UDP 19132.
- The former broadcaster account `[redacted-gamertag]` authenticated and created a session under Build 159, but logs later reported a nonce timeout that prevented joins. Its MCXboxBroadcast token cache was removed. Build 162 is now running and awaiting dedicated-account login.
- `HTTPS → :80` in exaroton is the web-service reverse-proxy route, not the Minecraft endpoint. Do not change it to a Minecraft port; Java and Bedrock use the delegated Minecraft domain and their own protocol ports.

## Last Reviewed

2026-11-09 — delegated custom DNS, migrated from Build 159 to Build 162, removed the personal auth cache and prepared dedicated-account login.


## Private source publication

This checkout contains the Compose deployment description. Device authentication, cache, player history, runtime configuration and backups are excluded. Review configuration and recovery before deployment. For updates review files individually, scan for secrets and private data, validate with nonsecret fixtures, push a fast-forward and verify the remote tree.


## Private GitHub repository (2026-11-17)

A private, reviewed source or configuration snapshot is available at `https://github.com/Noxist/mcxboxbroadcast-config` on `main`. The publication checkout is separate from the live working copy. Runtime data, secrets and other private material are excluded; any substituted placeholders and deployment limitations are documented in the repository. For updates review each changed file, run relevant isolated checks and secret scanning, push only a fast-forward, and verify the GitHub commit and tree. Do not overwrite or merge the live working history.

The publication snapshot redacts a historic personal gamertag; use the service account configured outside Git.
