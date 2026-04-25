# Recovery Notes

## Purpose

This document captures what the existing public Incredibuild lab images do before rebuilding or replacing them.

The goal is to preserve the useful all-in-one lab behavior while removing old assumptions, documenting the runtime clearly, and rebuilding from a cleaner source path.

## Images inspected

| Image | Tag | Notes |
|---|---|---|
| iwashuman2021/incredibuild | 20241110 | Historical base image. Appears to be a Debian 12 base with limited Incredibuild-specific content visible during initial inspection. |
| iwashuman2021/incredibuild | latest | Later image. Contains Incredibuild 4.13.0 installer/extracted payload and exposes ports 8080 and 8081, but default command is broken. |

## Inspection date

2026-04-24

## Local environment

- Host: StickerBox
- Runtime: Docker
- Repo path: `~/projects/incredibuild-lab`

## Local image sizes

| Image | Tag | Image ID | Disk usage | Content size |
|---|---|---|---|---|
| iwashuman2021/incredibuild | 20241110 | 223a4613910c | 5.41 GB | 1.05 GB |
| iwashuman2021/incredibuild | latest | 827c4bb14bd2 | 6.89 GB | 1.72 GB |

## Image metadata

### 20241110

- Image digest: `sha256:223a4613910c252f796fbdcfe42d6dc4605b9c44b4b382df853b4b0198b4fef2`
- Created: `2024-11-06T01:00:17.32559017+08:00`
- Entrypoint: `null`
- Cmd: `["incredibuild"]`
- Exposed ports: `null`

### latest

- Image digest: `sha256:827c4bb14bd20c2a3cbf7e157a0cf2e82e552d43d741a6dba5b624a16073882d`
- Created: `2024-11-11T08:43:45.343976184Z`
- Entrypoint: `null`
- Cmd: `["incredibuild"]`
- Exposed ports:
  - `8080/tcp`
  - `8081/tcp`

## Confirmed runtime issue

Both images define `["incredibuild"]` as the default command, but initial shell inspection did not find an `incredibuild` executable in `$PATH`.

Running `iwashuman2021/incredibuild:latest` normally fails with:

```text
exec: "incredibuild": executable file not found in $PATH

exit
cat > docs/recovery-notes.md <<'EOF'
# Recovery Notes

## Purpose

This document captures what the existing public Incredibuild lab images do before rebuilding or replacing them.

The goal is to preserve the useful all-in-one lab behavior while removing old assumptions, documenting the runtime clearly, and rebuilding from a cleaner source path.

## Images inspected

| Image | Tag | Notes |
|---|---|---|
| iwashuman2021/incredibuild | 20241110 | Historical base image. Appears to be a Debian 12 base with limited Incredibuild-specific content visible during initial inspection. |
| iwashuman2021/incredibuild | latest | Later image. Contains Incredibuild 4.13.0 installer/extracted payload and exposes ports 8080 and 8081, but default command is broken. |

## Inspection date

2026-04-24

## Local environment

- Host: StickerBox
- Runtime: Docker
- Repo path: `~/projects/incredibuild-lab`

## Local image sizes

| Image | Tag | Image ID | Disk usage | Content size |
|---|---|---|---|---|
| iwashuman2021/incredibuild | 20241110 | 223a4613910c | 5.41 GB | 1.05 GB |
| iwashuman2021/incredibuild | latest | 827c4bb14bd2 | 6.89 GB | 1.72 GB |

## Image metadata

### 20241110

- Image digest: `sha256:223a4613910c252f796fbdcfe42d6dc4605b9c44b4b382df853b4b0198b4fef2`
- Created: `2024-11-06T01:00:17.32559017+08:00`
- Entrypoint: `null`
- Cmd: `["incredibuild"]`
- Exposed ports: `null`

### latest

- Image digest: `sha256:827c4bb14bd20c2a3cbf7e157a0cf2e82e552d43d741a6dba5b624a16073882d`
- Created: `2024-11-11T08:43:45.343976184Z`
- Entrypoint: `null`
- Cmd: `["incredibuild"]`
- Exposed ports:
  - `8080/tcp`
  - `8081/tcp`

## Confirmed runtime issue

Both images define `["incredibuild"]` as the default command, but initial shell inspection did not find an `incredibuild` executable in `$PATH`.

Running `iwashuman2021/incredibuild:latest` normally fails with:

```text
exec: "incredibuild": executable file not found in $PATH
