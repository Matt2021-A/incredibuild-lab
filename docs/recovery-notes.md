# Recovery Notes

## Purpose

This document captures what the existing public Incredibuild lab images do before rebuilding or replacing them.

The goal is to preserve the useful all-in-one lab behavior while removing old assumptions, documenting the runtime clearly, and rebuilding from a cleaner source path.

## Images inspected

| Image | Tag | Notes |
|---|---|---|
| iwashuman2021/incredibuild | 20241110 | Historical base image. Appears to be a Debian 12 base with limited Incredibuild-specific content visible during initial shell inspection. |
| iwashuman2021/incredibuild | latest | Later image. Contains Incredibuild 4.13.0 installer/extracted payload and exposes ports 8080 and 8081, but the default command is broken and the install log shows the core install failed against a hard-coded Coordinator address. |

## Inspection date

2026-04-24 and 2026-04-25

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
- `PATH` includes LLVM tooling and standard system paths, but not the Incredibuild extracted payload path.

Observed `PATH`:

```text
/usr/lib/llvm-17/bin:/root/.cargo/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

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

Important directories under the extracted payload:

```text
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/bin
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/data
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/dev
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/etc
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/httpd
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/lib
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/licenses
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/samples
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/share
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
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/bin/ccache
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
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/create_sqlite_db.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/generate_keys.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/ib_restart.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/uninstall.sh
```

## Command behavior inside `latest`

The extracted payload is present, but it is not installed into the expected live runtime path.

Observed command behavior:

- `ib_console --help` works when called directly from the extracted payload path.
- `ib_test_coordinator` runs far enough to print usage when called without required arguments.
- `ib_version.sh` fails because `/opt/incredibuild/data/version_info.src` does not exist.
- `ib_info --help` fails because `/etc/incredibuild/log` does not exist.

This points to an incomplete install: the payload exists under `/opt/incredibuild_tmp`, but the expected runtime layout under `/opt/incredibuild` and `/etc/incredibuild` was not created successfully.

## Installer log findings

`/tmp/incredibuild-install.log` shows the installer ran in unattended mode and attempted an Initiator install against a hard-coded Coordinator address.

Key log findings:

```text
Preferred installation mode : unattended
coordinator: disabled
Core args: --data-dir /etc//ib_core --initiator --coordinator-machine=192.168.2.243 --license-type=SUVM
```

The dry run succeeded, but the real core install failed:

```text
ib_test_coordinator: Port 9953 is inaccessible, please check port value or disable firewall (connection refused).
ibsetup_linux_2097.ubin: Installation failed
```

The Coordinator test log confirms the failed connection attempt:

```text
Can't connect to 192.168.2.243:9953
```

Important nuance: the wrapper log ends with `Installation completed` and exits with code `0`, but the core installer failed earlier. That means the Docker build could have appeared successful while leaving the image in an incomplete runtime state.

## Early interpretation

The `latest` image is not empty. It includes a meaningful Incredibuild Linux payload, including Coordinator, server/agent, helper, console, and management components.

The current issue is not only the launch path. The install itself appears incomplete because it attempted to install as an Initiator pointed at a Coordinator that was unreachable at build time.

This is consistent with a lab image that was built around a specific local network and later pushed with those assumptions baked in.

The exposed ports `8080` and `8081` suggest the image was intended to surface UI or service endpoints, but no listener has been confirmed because services never start by default.

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
- Direct payload binaries can be called from `/opt/incredibuild_tmp/incredibuild_tmp-x86_64/bin`.
- Some management scripts expect completed runtime install paths that do not exist.

## Services and ports

The `latest` image exposes:

```text
8080/tcp
8081/tcp
```

No listener has been confirmed yet because the default startup path fails before services start.

The installer attempted to reach a Coordinator on port `9953` at build time. That connection failed.

## Incredibuild role behavior

What appears to be present in `latest`:

- Coordinator binary: present in extracted payload
- Server/agent binary: present in extracted payload
- Helper binary: present in extracted payload
- Build console binary: present in extracted payload
- Management scripts for Coordinator, server, helper, and UI-related behavior: present in extracted payload
- Samples directory: present in extracted payload

What is not yet confirmed:

- Whether these components can be activated in-place from the extracted payload
- Whether a fresh install can be completed inside a rebuilt image without relying on a pre-existing external Coordinator
- Whether the UI binds to ports 8080 or 8081
- Whether a sample build flow exists and can be run locally
- Whether lab-mode config can enable Coordinator, server, helper, and console roles together

## Hard-coded assumptions found

The installer log contains a hard-coded Coordinator address:

```text
192.168.2.243
```

The install attempted to use port:

```text
9953
```

This is likely an old local lab/network assumption. It should not be preserved in the rebuilt image.

## All-in-one behavior assessment

The `latest` image contains many of the ingredients for an all-in-one Incredibuild lab, but the actual build looks more like a failed Initiator install pointed at an external Coordinator.

That matters for positioning: the useful goal remains all-in-one, but the current public image may not actually achieve that goal in its default run state.

The recovery task should focus on turning the existing payload and recovered deployment intent into a predictable lab startup flow.

## Rebuild implications

The next clean rebuild should not rely on `CMD ["incredibuild"]`.

The rebuild also should not run an Initiator install against a fixed external Coordinator during image build.

Instead, it should use an explicit entrypoint script that:

1. Sets expected paths.
2. Verifies required binaries exist.
3. Either performs a clean local lab install at first run or uses a known-good installed runtime layout.
4. Starts required Incredibuild components in a known order.
5. Enables Coordinator, server/agent, helper, and UI behavior if lab mode supports it.
6. Prints useful status output.
7. Fails clearly if required files or runtime configuration are missing.
8. Keeps the container alive only after services are started.

## Suggested next inspection steps

1. Inspect the recovered Dockerfile and Kubernetes manifest to understand what command or runtime overrides were intended.
2. Inspect `ib_service.bash` to learn the service control model.
3. Inspect the `enable_*` and `disable_*` scripts to see whether they modify a config database or write simple flags.
4. Inspect the extracted `samples` folder.
5. Test whether `ib_coordinator`, `ib_server`, and `ib_helper` can print help/version output directly from the extracted payload.
6. Determine whether a clean rebuild should run the unified installer in Coordinator mode first, rather than Initiator mode pointed at an external Coordinator.

Do not commit raw inspection logs. Summarize findings here instead.

## Open questions

- Was `20241110` intended as only a compiler/base image?
- Was `latest` intended to be an Initiator image rather than all-in-one?
- Did Kubernetes override the broken default command in the original deployment?
- Which script should start Coordinator, server, helper, and UI roles?
- Are ports 8080 and 8081 the intended UI/API ports?
- Does `ib_service.bash` provide a usable startup path inside the container?
- Can `ib_test_coordinator` validate a local Coordinator once one is started?
- Does the lab need runtime-provided config before services can start?
- Can the installer complete in a local all-in-one mode without external Coordinator dependency?
- What did the recovered Kubernetes manifest override at runtime, if anything?
