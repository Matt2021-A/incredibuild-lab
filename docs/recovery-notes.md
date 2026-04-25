# Recovery Notes

## Purpose

This document captures what the existing public Incredibuild lab images do before rebuilding or replacing them.

The goal is to preserve the useful all-in-one lab behavior while removing old assumptions, documenting the runtime clearly, and rebuilding from a cleaner source path.

## Images inspected

| Image | Tag | Notes |
|---|---|---|
| iwashuman2021/incredibuild | 20241110 | Historical base image. Appears to be a Debian 12 base with limited Incredibuild-specific content visible during initial shell inspection. |
| iwashuman2021/incredibuild | latest | Later image. Contains Incredibuild 4.13.0 installer/extracted payload and exposes ports 8080 and 8081, but the default command is broken. |

## Inspection date

2026-04-24

## Local environment

- Host: StickerBox
- Runtime: Docker
- Repo path: `~/projects/incredibuild-lab`
- Branch: `docs/image-inspection`

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

Both images define `["incredibuild"]` as the default command, but shell inspection did not find an `incredibuild` executable in `$PATH`.

Running `iwashuman2021/incredibuild:latest` normally fails with:

```text
exec: "incredibuild": executable file not found in $PATH
```

This means the current default container run path is broken. The image can be inspected by overriding the entrypoint with a shell, but it does not currently behave as a pull-and-run lab image.

## Filesystem findings

### Base OS

Both inspected images report:

```text
Debian GNU/Linux 12 (bookworm)
```

### 20241110

Initial shell inspection found:

- `incredibuild` is not present in `$PATH`.
- No obvious installed Incredibuild runtime path was found in the first `find` pass.
- `/llvm.sh` exists.

Practical read: `20241110` may be closer to a compiler/base image than the final working all-in-one lab image.

### latest

Initial shell inspection found:

- `incredibuild` is not present in `$PATH`.
- Incredibuild 4.13.0 installer and extracted payload files are present.
- The image appears to contain the ingredients for the lab, but not a working launcher.

Important paths found:

```text
/root/incredibuild_4.13.0.run
/tmp/ib_installer_20241111084245
/tmp/incredibuild_4.13.0-2097.0
/tmp/incredibuild-install.log
/tmp/ib-install-logs/ib_test_coordinator.log
/opt/incredibuild_management
/opt/incredibuild_tmp
/opt/incredibuild_tmp/incredibuild_tmp-x86_64
```

Important binaries found under the extracted payload:

```text
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/bin/ib_console
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/bin/ib_coordinator
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/bin/ib_helper
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/bin/ib_info
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/bin/ib_server
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/bin/ib_submit
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/bin/ib_watchdog
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/bin/ib_docker
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/bin/ib_podman
```

Important management scripts found under the extracted payload:

```text
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/enable_coordinator.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/disable_coordinator.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/enable_server.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/disable_server.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/enable_helper.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/disable_helper.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/enable_httpd.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/disable_httpd.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/set_agent_params.py
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/get_agent_params.py
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/ib_service.bash
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/ib_version.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/manage_ui_access.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/ib_test_coordinator
```

## Early interpretation

The `latest` image is not empty. It includes a meaningful Incredibuild Linux payload, including Coordinator, server/agent, helper, console, and management components.

The current issue is the launch path. The image declares a default command named `incredibuild`, but no matching executable exists in `$PATH`.

This is consistent with the recovered Dockerfile pattern where a useful startup block may have been overwritten by a final `CMD ["incredibuild"]` instruction.

## Runtime behavior

### 20241110

- Shell override works.
- Base OS is Debian 12.
- `incredibuild` command is not present in `$PATH`.
- Normal run is expected to fail for the same reason as `latest`, but still needs explicit confirmation.

### latest

- Shell override works.
- Base OS is Debian 12.
- `incredibuild` command is not present in `$PATH`.
- Normal run fails immediately because Docker cannot execute `incredibuild`.

## Services and ports

The `latest` image exposes:

```text
8080/tcp
8081/tcp
```

No listener has been confirmed yet because the default startup path fails before services start.

## Incredibuild role behavior

What appears to be present in `latest`:

- Coordinator binary: present in extracted payload
- Server/agent binary: present in extracted payload
- Helper binary: present in extracted payload
- Build console binary: present in extracted payload
- Management scripts for Coordinator, server, helper, and UI-related behavior: present in extracted payload

What is not yet confirmed:

- Whether the payload was fully installed into a final runtime path
- Whether the services can start successfully from the extracted payload
- Whether the UI binds to ports 8080 or 8081
- Whether runtime configuration is required before startup
- Whether a sample build flow exists inside the image

## Hard-coded assumptions found

TBD after reviewing Docker history and sanitized recovered files.

## All-in-one behavior assessment

The `latest` image appears to contain the ingredients for an all-in-one Incredibuild lab, but not a working default launcher.

The recovery task should focus on turning the existing payload into a predictable lab startup flow.

## Rebuild implications

The next clean rebuild should not rely on `CMD ["incredibuild"]`.

Instead, it should use an explicit entrypoint script that:

1. Sets expected paths.
2. Verifies required binaries exist.
3. Starts required Incredibuild components in a known order.
4. Enables Coordinator, server/agent, helper, and UI behavior if needed.
5. Prints useful status output.
6. Fails clearly if required files or runtime configuration are missing.
7. Keeps the container alive only after services are started.

## Suggested next inspection steps

Inside `iwashuman2021/incredibuild:latest`, inspect the extracted payload directly:

```bash
export IBROOT=/opt/incredibuild_tmp/incredibuild_tmp-x86_64

ls -la /opt
ls -la /opt/incredibuild_tmp
ls -la "$IBROOT"
ls -la "$IBROOT/bin" | head -80
ls -la "$IBROOT/management" | head -120

"$IBROOT/management/ib_version.sh" || true
"$IBROOT/bin/ib_info" --help || true
"$IBROOT/bin/ib_console" --help || true
"$IBROOT/management/ib_test_coordinator" || true

sed -n '1,220p' "$IBROOT/management/ib_service.bash"
sed -n '1,220p' /tmp/incredibuild-install.log
sed -n '1,220p' /tmp/ib-install-logs/ib_test_coordinator.log
```

Do not commit raw inspection logs. Summarize findings here instead.

## Open questions

- Was `20241110` intended as only a compiler/base image?
- Was `latest` intended to install Incredibuild fully, or only unpack the installer?
- Which script should start Coordinator, server, helper, and UI roles?
- Are ports 8080 and 8081 the intended UI/API ports?
- Does `ib_service.bash` provide a usable startup path inside the container?
- Can `ib_test_coordinator` validate the Coordinator inside the container?
- Does the lab need runtime-provided config before services can start?
- What did the recovered Kubernetes manifest override at runtime, if anything?
