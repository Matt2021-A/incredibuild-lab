# Service and Samples Inspection

## Purpose

This document captures what the Incredibuild extracted payload reveals about service control and bundled sample workflows.

Inspection was performed inside:

```text
iwashuman2021/incredibuild:latest
```

with:

```bash
IBROOT=/opt/incredibuild_tmp/incredibuild_tmp-x86_64
```

## Service script inspected

```text
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/ib_service.bash
```

Only the first portion of the script was inspected in this pass.

## `ib_service.bash` findings

The service script is a cross-init-system service installer/activator. It detects the host init model and installs or removes service definitions accordingly.

Supported init types observed:

```text
systemd
openrc
sysv
darwin
```

Important behavior:

- Detects Linux versus Darwin.
- On Linux, inspects `/proc/1/exe` to determine whether PID 1 is systemd, openrc, or something else.
- Falls back to sysv if systemd/openrc are not detected.
- For systemd, installs an `incredibuild.service` unit and runs `systemctl enable` plus `daemon-reload`.
- For sysv, installs an init script and creates rc links.
- For openrc, installs an openrc service and registers it with `rc-update`.
- For Darwin, installs a launch plist.

## Container implication

This service model expects a traditional init environment.

A normal Docker container usually does not run systemd as PID 1. In the inspected shell session, the container was entered through `/bin/bash`, so the script would not have a normal systemd service manager available.

Practical implication: the rebuilt lab image should not rely on `systemctl enable` or normal boot-time service registration as the primary runtime path.

The lab entrypoint should either:

1. Start the required Incredibuild binaries directly in the foreground/background, or
2. Run a minimal process supervisor if the product requires multiple long-running processes, or
3. Explicitly support a systemd-style container only if there is a strong reason.

Default recommendation: avoid systemd-in-container unless testing proves the binaries cannot run cleanly without it.

## Enable and disable script findings

The enable/disable scripts are thin wrappers around shared functions in:

```text
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/functions.src
```

Each script sources `functions.src` and calls either `activate_service` or `deactivate_service` for a named component.

Observed mappings:

| Script | Action |
|---|---|
| `enable_coordinator.sh` | `activate_service incredibuild_coordinator` |
| `disable_coordinator.sh` | `deactivate_service incredibuild_coordinator` |
| `enable_server.sh` | `activate_service incredibuild_server` |
| `disable_server.sh` | `deactivate_service incredibuild_server` |
| `enable_helper.sh` | `activate_service incredibuild_helper` |
| `disable_helper.sh` | `deactivate_service incredibuild_helper` |
| `enable_httpd.sh` | `activate_service incredibuild_httpd` |
| `disable_httpd.sh` | `deactivate_service incredibuild_httpd` |

## Interpretation

The role toggles do not directly start binaries in the inspected snippets. They delegate to `functions.src`, which likely manages service registration, service metadata, config layout, symlinks, or runtime database state.

The next required inspection target is:

```text
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/functions.src
```

Specifically, inspect:

- `activate_service`
- `deactivate_service`
- service source/destination paths
- references to `incredibuild_coordinator`
- references to `incredibuild_server`
- references to `incredibuild_helper`
- references to `incredibuild_httpd`

## Samples directory findings

Bundled samples exist under:

```text
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/samples
```

Observed sample tree:

```text
samples/
├── custom_execution/
│   ├── README
│   ├── ib_profile.xml
│   └── ib_test_invoker
└── make_build/
    ├── Makefile
    ├── README
    ├── hello.c
    └── ib_profile.xml
```

## Sample interpretation

The `make_build` sample is the best candidate for the rebuilt lab's default demo path.

The recovered Dockerfile previously cloned an external sample repo and ran a sample build from there. That is not ideal for a self-contained lab image.

Better rebuild direction:

- Use the bundled `samples/make_build` sample first.
- Copy or expose the sample path in documentation.
- Gate demo execution behind `IB_RUN_SAMPLE=true`.
- Keep default startup focused on starting the lab, not running a build automatically.

## Rebuild implication

The all-in-one lab should treat the Incredibuild service model and the sample build as separate concerns.

Startup should:

1. Install or activate the runtime.
2. Start the required local services.
3. Print status and next commands.

Sample execution should be optional:

```bash
ib_console -f -p samples/make_build/ib_profile.xml --no-cgroups make -C samples/make_build
```

The exact sample command must be validated before publication.

## Next inspection commands

Run inside the image:

```bash
export IBROOT=/opt/incredibuild_tmp/incredibuild_tmp-x86_64

printf '===== functions.src key functions =====\n'
grep -n "activate_service\|deactivate_service\|incredibuild_coordinator\|incredibuild_server\|incredibuild_helper\|incredibuild_httpd" "$IBROOT/management/functions.src"

printf '===== functions.src first 260 lines =====\n'
sed -n '1,260p' "$IBROOT/management/functions.src"

printf '===== make_build README =====\n'
sed -n '1,220p' "$IBROOT/samples/make_build/README"

printf '===== make_build Makefile =====\n'
sed -n '1,220p' "$IBROOT/samples/make_build/Makefile"

printf '===== make_build profile =====\n'
sed -n '1,220p' "$IBROOT/samples/make_build/ib_profile.xml"
```

Do not commit raw output unless reviewed. Summarize findings in this document or in `recovery-notes.md`.
