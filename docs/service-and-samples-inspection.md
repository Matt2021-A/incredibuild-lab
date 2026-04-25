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

## `functions.src` findings

`functions.src` is much simpler than expected.

It defines four helper functions:

```text
link_opt_2_ib_init_d
unlink_opt_2_ib_init_d
activate_service
deactivate_service
```

Observed behavior:

- `link_opt_2_ib_init_d <service>` creates a symlink from `/opt/incredibuild/etc/init.d/<service>` to `/etc/incredibuild/init.d/<service>`.
- `unlink_opt_2_ib_init_d <service>` removes the symlink from `/etc/incredibuild/init.d/<service>`.
- `activate_service <service>` creates the symlink and, by default, starts the service through `/etc/incredibuild/init.d/<service> start`.
- `deactivate_service <service>` stops the service through `/opt/incredibuild/etc/init.d/<service> stop` if a pidfile exists, then removes the `/etc/incredibuild/init.d/<service>` symlink.

Important environment behavior:

```text
INCREDIBUILD_SERVICE_START defaults to yes.
```

If `INCREDIBUILD_SERVICE_START` is set to a value other than `yes`, `activate_service` will create the symlink but will not start the service.

## Service model interpretation

The enable/disable scripts are not database mutations. They are symlink-and-start wrappers around installed init scripts.

That means they require a completed install layout with these paths present:

```text
/opt/incredibuild/etc/init.d/<service>
/etc/incredibuild/init.d/<service>
```

The current `latest` image does not appear to have a completed install layout. The extracted payload is under:

```text
/opt/incredibuild_tmp/incredibuild_tmp-x86_64
```

But the scripts expect live paths under:

```text
/opt/incredibuild
/etc/incredibuild
```

This confirms that the current public `latest` image contains the extracted installer payload but not a completed runnable installation.

## Rebuild implication from service model

The rebuilt image needs a real install phase before these scripts are useful.

The entrypoint should not call `enable_coordinator.sh`, `enable_server.sh`, `enable_helper.sh`, or `enable_httpd.sh` until it has verified that these exist:

```text
/opt/incredibuild/etc/init.d/incredibuild_coordinator
/opt/incredibuild/etc/init.d/incredibuild_server
/opt/incredibuild/etc/init.d/incredibuild_helper
/opt/incredibuild/etc/init.d/incredibuild_httpd
/etc/incredibuild/init.d
```

If those files are missing, the image should either perform first-run install or fail clearly.

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

## `make_build` sample findings

The `make_build` README states that the sample demonstrates remote distribution and acceleration of tasks generated by the `make` build tool.

The documented sample command is:

```bash
ib_console make -j30
```

The Makefile defines 30 phony tasks and compiles `hello.c` for each task:

```makefile
LIST := 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30

all: $(LIST)

.PHONY: $(LIST)

$(LIST):
	gcc hello.c -DI=$@ -c -o /dev/null
```

The profile marks `make` as intercepted and `gcc` as local-only for compile-only style arguments:

```xml
<process filename="make" type="intercepted" />
<process filename="gcc"  type="local_only" exclude_args="-c:-S:-E" />
```

The profile also sets:

```xml
monitor_enable="false"
requested_cores="10000"
```

## Sample interpretation

The `make_build` sample is the best candidate for the rebuilt lab's optional demo path.

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
2. Verify `/opt/incredibuild` and `/etc/incredibuild` exist.
3. Start the required local services.
4. Print status and next commands.

Sample execution should be optional:

```bash
cd /opt/incredibuild/samples/make_build
ib_console make -j30
```

The exact sample path depends on where the completed install places samples. If the installer does not copy samples into `/opt/incredibuild/samples`, the rebuild can copy them into a stable lab path such as:

```text
/opt/incredibuild-lab/samples/make_build
```

## Updated technical conclusion

The service toggles are usable only after a completed install exists.

The current public image failed before reaching that state. This confirms the rebuild should use a first-run install model, with clear runtime validation and visible failure messages.

## Next inspection commands

Run inside the image to inspect installer help and determine the correct local Coordinator or all-in-one install flags:

```bash
/root/incredibuild_4.13.0.run --help || true
/tmp/ib_installer_20241111084245/ibsetup_linux_2097.ubin --help || true
/tmp/ib_installer_20241111084245/ibsetup_linux_2097.ubin install --help || true
```

Also inspect the init scripts if a completed install can be produced in a fresh container or rebuilt image:

```bash
ls -la /opt/incredibuild/etc/init.d
sed -n '1,180p' /opt/incredibuild/etc/init.d/incredibuild_coordinator
sed -n '1,180p' /opt/incredibuild/etc/init.d/incredibuild_server
sed -n '1,180p' /opt/incredibuild/etc/init.d/incredibuild_helper
sed -n '1,180p' /opt/incredibuild/etc/init.d/incredibuild_httpd
```

Do not commit raw output unless reviewed. Summarize findings in this document or in `recovery-notes.md`.
