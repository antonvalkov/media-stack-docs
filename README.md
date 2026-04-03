# Media Stack Deployment And Operations Guide

This guide documents the final working shape of a LAN-only media stack built around `qBittorrent`, `Sonarr`, `Radarr`, `Prowlarr`, `Bazarr`, and `Lidarr`.

It is written from two sources:

- saved project history
- current live runtime validation

This public version is sanitized. Replace placeholders such as `<MEDIA_VM_IP>` and `<NAS_MEDIA_EXPORT>` with your own values.

## What This Stack Does

The stack uses one Debian VM to run all media services in Docker. Media files live on a NAS and are mounted into the VM. Downloads also land on NAS-backed storage so the apps can import and move files without copying across machines.

This design was chosen after an earlier LXC attempt was abandoned. In this project, direct NFS use inside an unprivileged container was the wrong fit. A VM gave a cleaner and more predictable storage model.

## Final Architecture

### Platform Layout

- Hypervisor: Proxmox
- Guest type: Debian 12 VM
- Runtime: Docker Compose
- Network model: LAN-only
- Reverse proxy: none required
- VPN: none required for the base deployment

### Service Roles

- `qBittorrent`: download client and torrent session manager
- `Sonarr`: TV series management and episode imports
- `Radarr`: movie management and imports
- `Prowlarr`: central indexer manager and app sync layer
- `Bazarr`: subtitle management
- `Lidarr`: music management

### Default Service Ports

| Service | Port |
| --- | --- |
| qBittorrent Web UI | `8080` |
| qBittorrent peer traffic | `6881/tcp` and `6881/udp` |
| Sonarr | `8989` |
| Prowlarr | `9696` |
| Radarr | `7878` |
| Bazarr | `6767` |
| Lidarr | `8686` |

### High-Level Data Flow

1. Add indexers in `Prowlarr`.
2. Sync `Prowlarr` with `Sonarr` and `Radarr`.
3. Search from `Sonarr` or `Radarr`.
4. Send the selected release to `qBittorrent`.
5. Let `qBittorrent` download into the NAS-backed downloads path.
6. Let `Sonarr` or `Radarr` import from the completed download path into the media library.
7. Let `Bazarr` scan the resulting library for subtitles.

## Final Storage And Path Model

This was the most important operational lesson in the project. Treat path parity as mandatory.

### NAS And NFS Mounts

Use two NAS-backed sources:

- a media export for the library
- a downloads export for temporary and completed downloads

Final host-side model:

- NAS media export: `<NAS_MEDIA_EXPORT>` mounted at `/mnt/media`
- NAS downloads export: `<NAS_DOWNLOADS_EXPORT>` mounted at `/mnt/downloads-source`
- bind mount from `/mnt/downloads-source/<DOWNLOADS_SUBPATH>` to `/mnt/downloads/sonartmp`

Application-visible VM paths:

- movies: `/mnt/media/Movies`
- series: `/mnt/media/Series`
- music: `/mnt/media/Music`
- downloads: `/mnt/downloads/sonartmp`

### Container Path Layout

Final live container mount model:

- `qBittorrent`
  - VM `/mnt/downloads/sonartmp` -> container `/downloads`
- `Sonarr`
  - VM `/mnt/media` -> container `/mnt/media`
  - VM `/mnt/downloads` -> container `/mnt/downloads`
- `Radarr`
  - VM `/mnt/media` -> container `/mnt/media`
  - VM `/mnt/downloads` -> container `/mnt/downloads`
- `Lidarr`
  - VM `/mnt/media` -> container `/mnt/media`
  - VM `/mnt/downloads` -> container `/mnt/downloads`
- `Bazarr`
  - VM `/mnt/media` -> container `/mnt/media`
- `Prowlarr`
  - config-only mount

### Why This Matters

The downloader and the importing apps do not currently see the same path string.

- `qBittorrent` reports completed items under `/downloads/...`
- `Sonarr`, `Radarr`, and `Lidarr` see the same files under `/mnt/downloads/sonartmp/...`

If you ignore that mismatch, imports fail even when the file exists on disk.

## Docker Compose Layout

Recommended layout:

- compose file: `/opt/media-stack/docker-compose.yml`
- config root: `/opt/media-stack/config`

Per-service config directories:

- `/opt/media-stack/config/qbittorrent`
- `/opt/media-stack/config/sonarr`
- `/opt/media-stack/config/prowlarr`
- `/opt/media-stack/config/radarr`
- `/opt/media-stack/config/bazarr`
- `/opt/media-stack/config/lidarr`

Helpful downloads subdirectories:

- `/mnt/downloads/sonartmp/complete`
- `/mnt/downloads/sonartmp/incomplete`
- `/mnt/downloads/sonartmp/watch`

## Bazarr Bulgarian Subtitles

The stack was later extended with a Bulgarian subtitle workflow in `Bazarr`.

### Working Bazarr Pattern

- `Bazarr` linked to both `Sonarr` and `Radarr`
- desired language enabled: `bg`
- language profile created: `Bulgarian`
- default subtitle profile enabled for:
  - series
  - movies

Use a single-language profile if your goal is simply "download Bulgarian subtitles where available".

### Recommended Setup Shape

1. Link `Bazarr` to `Sonarr` and `Radarr`.
2. Enable `Bulgarian` as a desired language.
3. Create a language profile named `Bulgarian`.
4. Set it as the default profile for series and movies.
5. Assign the profile to existing synced series or movies.
6. Run a full subtitle scan once.
7. Run a wanted-subtitles search.

### Runtime Validation Pattern

After setup, verify:

- `Bazarr` health is clean
- the language profile exists
- synced series show the expected `profileId`
- missing subtitle counts look plausible after a full scan

## Bazarr Providers: Saved Config Versus Active Runtime

One important lesson from runtime validation was that saved provider keys and active provider runtime do not always match.

Do not assume a provider is truly active only because its key appears in saved settings.

Safer rule:

- saved config shows what `Bazarr` remembers
- runtime provider views show what `Bazarr` is actually exposing and attempting to use

When documenting or troubleshooting providers, treat the runtime provider views as authoritative.

## Addic7ed Setup

`Addic7ed` support exists in `Bazarr`, but in practice it may require more than just username and password.

### Common Working Inputs

- username
- password
- logged-in browser cookies
- matching browser user-agent

In some cases, an anti-captcha provider may also be required. For a home LAN-only stack, cookies plus user-agent are often the more practical first choice.

### Safe Procedure

1. Log into `Addic7ed` in a normal browser.
2. Open browser developer tools.
3. Copy the logged-in cookies for the site.
4. Copy the request user-agent from the same browser.
5. Paste those values into the `Bazarr` `Addic7ed` provider settings.
6. Enable the provider and validate it in `Bazarr`.

### Security Rule

Never store real cookies, session IDs, usernames, passwords, API keys, or user-agent-derived secret material in public docs, git history, screenshots meant for publication, or commits.

Document only the process, not the live values.

## Unsupported Or Non-Native Bulgarian Subtitle Sites

Do not assume every Bulgarian subtitle site can be added to `Bazarr`.

Examples that should not be treated as verified native Bazarr providers unless runtime proves otherwise:

- `subs.sab.bz`
- `jordansl.free.bg`

Practical rule:

- if a site is not a native provider in your current Bazarr build, do not document it as part of the supported automation path
- keep the docs aligned with what the running Bazarr instance can really use

## Import-Path Mismatch Is Still The Main Media Import Failure Mode

This remained the most important operational lesson even after subtitle automation was added.

### Current Translation Model

- qBittorrent reports completed files as `/downloads/complete/...`
- Sonarr, Radarr, and Lidarr see the same files at `/mnt/downloads/sonartmp/complete/...`

If the importing application is not translating those paths through a remote path mapping, the import fails even when the file is already present on disk.

### Why Subtitle Work Is Different

This import-path mismatch is mainly a `Sonarr` / `Radarr` / `Lidarr` media-import problem.

`Bazarr` normally works from the imported library paths under:

- `/mnt/media/Series/...`
- `/mnt/media/Movies/...`

So when subtitles fail, look first at:

- provider support
- credentials or cookies
- language profile
- score rules
- subtitle availability

Do not jump straight to the `/downloads/...` mapping unless the media import itself is failing.

## Step-By-Step Deployment

### 1. Create The VM

Build one Debian VM for the whole stack.

Recommended baseline:

- 2 vCPU
- 4 GB RAM
- 20 GB system disk
- static LAN address
- Docker-capable Debian install

Install:

- `docker`
- `docker-compose` or Compose plugin
- `nfs-common`
- `qemu-guest-agent`

### 2. Mount NAS Storage In The VM

Create the mount points:

```bash
mkdir -p /mnt/media
mkdir -p /mnt/downloads-source
mkdir -p /mnt/downloads/sonartmp
```

Add persistent mounts in `/etc/fstab` using your own values:

```fstab
<NAS_MEDIA_EXPORT> /mnt/media nfs4 vers=4.1,_netdev,x-systemd.automount,noatime 0 0
<NAS_DOWNLOADS_EXPORT> /mnt/downloads-source nfs4 vers=4.1,_netdev,x-systemd.automount,noatime 0 0
/mnt/downloads-source/<DOWNLOADS_SUBPATH> /mnt/downloads/sonartmp none bind,_netdev,x-systemd.requires=/mnt/downloads-source 0 0
```

Validate:

```bash
findmnt /mnt/media /mnt/downloads-source /mnt/downloads/sonartmp
ls -ld /mnt/media /mnt/media/Movies /mnt/media/Series /mnt/media/Music
ls -ld /mnt/downloads /mnt/downloads/sonartmp
```

### 3. Create The Compose Stack

Create `/opt/media-stack/docker-compose.yml`.

Use this structure:

```yaml
version: "3.8"
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      PUID: "<PUID>"
      PGID: "<PGID>"
      TZ: "<TIMEZONE>"
      WEBUI_PORT: "8080"
      UMASK: "002"
    volumes:
      - /opt/media-stack/config/qbittorrent:/config
      - /mnt/downloads/sonartmp:/downloads
    ports:
      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      PUID: "<PUID>"
      PGID: "<PGID>"
      TZ: "<TIMEZONE>"
      UMASK: "002"
    volumes:
      - /opt/media-stack/config/sonarr:/config
      - /mnt/media:/mnt/media
      - /mnt/downloads:/mnt/downloads
    ports:
      - "8989:8989"
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      PUID: "<PUID>"
      PGID: "<PGID>"
      TZ: "<TIMEZONE>"
      UMASK: "002"
    volumes:
      - /opt/media-stack/config/prowlarr:/config
    ports:
      - "9696:9696"
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      PUID: "<PUID>"
      PGID: "<PGID>"
      TZ: "<TIMEZONE>"
      UMASK: "002"
    volumes:
      - /opt/media-stack/config/radarr:/config
      - /mnt/media:/mnt/media
      - /mnt/downloads:/mnt/downloads
    ports:
      - "7878:7878"
    restart: unless-stopped

  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    environment:
      PUID: "<PUID>"
      PGID: "<PGID>"
      TZ: "<TIMEZONE>"
      UMASK: "002"
    volumes:
      - /opt/media-stack/config/bazarr:/config
      - /mnt/media:/mnt/media
    ports:
      - "6767:6767"
    restart: unless-stopped

  lidarr:
    image: lscr.io/linuxserver/lidarr:latest
    container_name: lidarr
    environment:
      PUID: "<PUID>"
      PGID: "<PGID>"
      TZ: "<TIMEZONE>"
      UMASK: "002"
    volumes:
      - /opt/media-stack/config/lidarr:/config
      - /mnt/media:/mnt/media
      - /mnt/downloads:/mnt/downloads
    ports:
      - "8686:8686"
    restart: unless-stopped

  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      TZ: "<TIMEZONE>"
    ports:
      - "8191:8191"
    restart: unless-stopped
```

Start the stack:

```bash
cd /opt/media-stack
docker-compose up -d
```

### 4. Configure qBittorrent

Initial tasks:

1. Sign in to the Web UI.
2. Change the temporary first-run password.
3. Confirm download folders point into `/downloads`.
4. Keep completed downloads under `/downloads/complete`.
5. Keep incomplete downloads under `/downloads/incomplete`.

Important note:

If the image prints a `chown` warning against NAS-backed storage, do not assume the stack is broken. On this project, the warning was expected and did not block download writes.

### 5. Configure Sonarr

Set:

- root folder: `/mnt/media/Series`
- download client: `qBittorrent`
- category: a dedicated TV category if you use one
- completed download handling: enabled

Do not blindly add remote path mappings on day one. First verify whether the downloader and importer already share a valid path model.

### 6. Configure Radarr And Lidarr

Set:

- `Radarr` root folder: `/mnt/media/Movies`
- `Lidarr` root folder: `/mnt/media/Music`
- same `qBittorrent` instance as the download client
- completed download handling enabled where applicable

### 7. Configure Prowlarr

Add your indexers in `Prowlarr`, then connect:

- `Sonarr`
- `Radarr`

This project used `Prowlarr` as the central indexer manager because it is cleaner than duplicating indexer setup across apps.

Operational lesson from runtime:

- some indexers may exist in `Prowlarr` definitions but still fail at runtime because of upstream blocking or anti-bot protection
- validate each indexer in the current runtime instead of assuming support means usability

### 8. Validate The Whole Stack

Check containers:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Check path visibility from inside the containers:

```bash
docker exec qbittorrent ls -ld /downloads /downloads/complete /downloads/incomplete
docker exec sonarr ls -ld /mnt/downloads /mnt/downloads/sonartmp /mnt/media/Series
docker exec radarr ls -ld /mnt/downloads /mnt/media/Movies
docker exec lidarr ls -ld /mnt/downloads /mnt/media/Music
```

Check local HTTP reachability:

```bash
for p in 8080 8989 9696 7878 6767 8686; do
  code=$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:$p/ || true)
  echo "$p:$code"
done
```

Then perform one real end-to-end test:

1. add a small test item in `Sonarr` or `Radarr`
2. search manually
3. send the release to `qBittorrent`
4. confirm the file lands in the completed downloads folder
5. confirm the importing app can see the same file path it was told to import
6. confirm the media file appears under the final library root

## Operational Lessons Learned

### Import-Path Mapping Lesson

This was the main failure mode in the project.

Observed live behavior:

- `qBittorrent` downloaded successfully
- `Sonarr` sent releases successfully
- `Sonarr` then failed import with `path does not exist or is not accessible`

Root cause:

- `qBittorrent` reported the completed path as `/downloads/complete/...`
- `Sonarr` only had direct visibility into the same file as `/mnt/downloads/sonartmp/complete/...`

Live-runtime note:

When this guide was written, that mismatch still existed in the active compose stack and Sonarr logs were showing repeated import failures against `/downloads/complete/...`.

That means the file existed on the VM, but the path string returned to `Sonarr` was not valid inside the `Sonarr` container.

Use this path map whenever troubleshooting:

| Layer | Path |
| --- | --- |
| NAS visible path | `<NAS_DOWNLOADS_EXPORT>/<DOWNLOADS_SUBPATH>/complete/...` |
| VM real path | `/mnt/downloads/sonartmp/complete/...` |
| qBittorrent container sees | `/downloads/complete/...` |
| Sonarr or Radarr container sees | `/mnt/downloads/sonartmp/complete/...` |
| Download client reports | `/downloads/complete/...` |

Safe fix options:

1. make downloader and importer use the same in-container path
2. or add a correct remote path mapping that translates `/downloads` to `/mnt/downloads/sonartmp`

Do not guess. Prove the path from:

- the VM host
- the downloader container
- the importing container

### Bulk Search Vs Manual Search Lesson

Manual episode search can work while season or series search still downloads nothing. That is not automatically a broken setup.

Observed live behavior:

- episode searches downloaded specific missing episodes
- season search processed releases but downloaded `0`
- series search processed releases but downloaded `0`

Why that happened:

- manual search targeted a single missing episode and matched episode-level releases
- season or series search mostly surfaced full-season packs
- Sonarr rejected those packs for valid reasons seen in debug logs, including:
  - `Full season pack`
  - `Release in queue already meets cutoff`
  - `Existing file meets cutoff`
  - quality or profile rejections for unwanted formats

Practical rule:

Before changing Sonarr settings, check whether the problem is really configuration or simply the shape of the releases being returned by the indexers.

## Troubleshooting

### When Imports Fail

Check:

1. the exact path reported by the download client
2. whether that exact path exists inside the importing container
3. the importing app logs
4. container mount parity

Helpful checks:

```bash
docker exec qbittorrent ls -l /downloads/complete
docker exec sonarr ls -l /mnt/downloads/sonartmp/complete
docker exec sonarr sh -lc 'tail -n 100 /config/logs/sonarr.txt'
docker inspect qbittorrent sonarr --format '{{.Name}} {{range .Mounts}}{{.Source}}=>{{.Destination}} {{end}}'
```

### When Searches Return Zero Downloads

Check:

1. the series or movie is monitored
2. missing items really exist in the wanted list
3. the indexers are synced from `Prowlarr`
4. the debug logs for rejection reasons

Helpful checks:

```bash
docker exec sonarr sh -lc "grep -RniE 'Full season pack|meets cutoff|not wanted in profile' /config/logs/sonarr.debug*.txt | tail -n 50"
docker exec prowlarr sh -lc 'tail -n 100 /config/logs/prowlarr.txt'
```

### When Indexers Seem Present But Do Not Work

Do not stop at "the indexer exists in Prowlarr".

Check:

1. runtime test result in `Prowlarr`
2. whether the indexer is TV-only or movie-only
3. anti-bot or Cloudflare failures
4. whether the app sync completed

## Validation Checklist

- VM is reachable
- NFS mounts are present after boot
- bind mount is present after boot
- all containers are up
- each app can see its media root
- `qBittorrent` can write to the downloads path
- `Prowlarr` can talk to `Sonarr` and `Radarr`
- one real manual search succeeds
- one real import succeeds
- reboot test succeeds

Do not mark the stack complete until the import step has succeeded with real files.

## Updates And Routine Operations

Update OS packages:

```bash
apt-get update && apt-get upgrade -y
```

Update container images:

```bash
cd /opt/media-stack
docker-compose pull
docker stop flaresolverr
docker rm flaresolverr
docker-compose up -d flaresolverr
docker-compose up -d
```

The explicit `stop` and `rm` step for `flaresolverr` is required when migrating from an older manually created container to the new Compose-managed service. Without that cleanup, Docker reports a container name conflict for `/flaresolverr`.

Check status after updates:

```bash
docker ps
findmnt /mnt/media /mnt/downloads-source /mnt/downloads/sonartmp
```

## Rollback Notes

If a compose change breaks the stack:

1. restore the previous `docker-compose.yml`
2. run `docker-compose up -d`
3. re-check container mounts and app reachability

If the `flaresolverr` Compose migration causes trouble:

1. restore the previous `docker-compose.yml`
2. remove the Compose-managed `flaresolverr` container
3. recreate the old standalone `flaresolverr` container only if you still need the pre-Compose state

If a mount change breaks imports:

1. restore the prior `/etc/fstab`
2. remount the affected paths
3. validate the path map from host and containers before retrying imports

If a container update breaks behavior:

1. pull the previous image tag if you track tags explicitly
2. restart only the affected service
3. re-run one real search and one real import test

Avoid destructive rollback steps such as deleting media, removing qBittorrent state, or wiping app config unless you have a confirmed backup.

## Public Repo Sanitization Notes

This public-ready guide intentionally replaces:

- internal IP addresses
- NAS export paths
- hostnames
- user-specific download subpaths
- service credentials
- API keys
- tokens
- passwords
- private keys

Keep that policy if you publish screenshots, compose files, logs, or follow-up notes.
