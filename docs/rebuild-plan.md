# Rebuild Plan

## Decision

Use a first-run install model if runtime configuration is required.

The original image attempted to install Incredibuild during Docker build as an Initiator pointed at a fixed external Coordinator. That made the image dependent on an old local network state and allowed Docker build to appear successful even though the core install failed.

The rebuilt lab image should avoid that pattern.

## Why first-run install

A first-run install model is a better fit for this project because the lab needs to be configurable at runtime.

The image should contain the installer, helper scripts, and sample assets. The container should perform setup during startup only after runtime values are known.

This allows the lab to support two paths:

1. Local all-in-one lab mode.
2. External Coordinator mode, if someone intentionally wants that later.

The default should be local lab mode, because the purpose of this project is to reduce first-run friction.

## Installer capability findings

Installer help confirms the unified installer supports multiple roles directly:

```text
--coordinator enabled|disabled
--secondary-coordinator enabled|disabled
--initiator enabled|disabled
--helper enabled|disabled
--build-cache-service enabled|disabled
```

The lower-level `ibsetup_linux_2097.ubin install` help confirms shorthand role flags:

```text
-C, --coordinator             primary Coordinator activation
-E, --secondary-coordinator   secondary Coordinator activation
-I, --initiator               Initiator activation
-H, --helper                  Helper activation
-S                            Initiator and Helper activation, alias to -I -H
-Y, --build-cache-service     Build Cache Service
```

Mandatory install options:

```text
-A, --data-dir=<path>
-O, --coordinator-machine=<coordinator> required only when -C is not given
```

Important defaults:

```text
-G, --local-http-port=<port>  default 8080
-L, --local-https-port=<port> default 8081
-N, --utility-port=<port>     default 9953
-R, --license-type=<type>     DEFAULT or SUVM
```

Key implication: a Coordinator install does not require `--coordinator-machine`. That gives the rebuild a clean local-first path.

## Data directory validation finding

A first validation attempt used:

```text
--data-dir /etc/incredibuild
```

The installer rejected it with:

```text
The data directory cannot be /etc/incredibuild
```

No install layout was created after that failure:

```text
/opt/incredibuild             missing
/etc/incredibuild             missing
/opt/incredibuild/etc/init.d  missing
/etc/incredibuild/init.d      missing
```

A second validation attempt used:

```text
--data-dir /var/lib/incredibuild
```

That path was accepted, and the install completed far enough to create the expected runtime layout.

## Successful first-run install validation

This command completed inside a disposable container:

```bash
/root/incredibuild_4.13.0.run \
  --action install \
  --accept-eula true \
  --coordinator enabled \
  --initiator enabled \
  --helper enabled \
  --data-dir /var/lib/incredibuild \
  --local-http-port 8080 \
  --local-https-port 8081 \
  --utility-port 9953 \
  --license-type SUVM \
  --disable-telemetry true
```

Installer output included:

```text
Starting incredibuild core install
IncrediBuild Single-Use VM Agent (SUVM) installed
Starting incredibuild manager install
Incredibuild manager installation ...
Incredibuild manager installation: done
Saving parameters
Starting incredibuild manager services
Service 'incredibuild-manager-service' not found.
Service 'incredibuild-message-broker-service' not found.
Incredibuild installation: done
```

The manager service warnings did not stop the install from creating the core runtime layout. They should be tracked separately during service validation.

Created runtime paths:

```text
/opt/incredibuild
/etc/incredibuild -> /var/lib/incredibuild/ib_core
/var/lib/incredibuild/ib_core
/var/lib/incredibuild/ib_manager
/opt/incredibuild/etc/init.d
/etc/incredibuild/init.d
```

Important install markers and config files found under `/opt/incredibuild`:

```text
.done
.ib_do_not_start_services_after_install
.ib_install_command
.ib_storage_dir
```

The `.ib_do_not_start_services_after_install` marker is important. The entrypoint should not assume the installer leaves all services running.

Expected init scripts were created under `/opt/incredibuild/etc/init.d`, including:

```text
incredibuild
incredibuild_babysit
incredibuild_coordinator
incredibuild_dataaccess
incredibuild_helper
incredibuild_httpd
incredibuild_info
incredibuild_secstorage
incredibuild_server
incredibuild_telemetry
incredibuild_upgrade
incredibuild_watchdog
```

Symlinks were created under `/etc/incredibuild/init.d`, including:

```text
incredibuild_babysit
incredibuild_coordinator
incredibuild_dataaccess
incredibuild_helper
incredibuild_httpd
incredibuild_info
incredibuild_server
incredibuild_watchdog
```

This validates the core rebuild assumption: local all-in-one first-run install is technically viable using `/var/lib/incredibuild` as the data root.

## Proposed install strategy

Default lab mode should perform a first-run Coordinator plus local worker install using `/var/lib/incredibuild` as the default data root.

The first-run install command shape is now:

```bash
/root/incredibuild_4.13.0.run \
  --action install \
  --accept-eula true \
  --coordinator enabled \
  --initiator enabled \
  --helper enabled \
  --data-dir "$IB_DATA_ROOT" \
  --local-http-port "$IB_LOCAL_HTTP_PORT" \
  --local-https-port "$IB_LOCAL_HTTPS_PORT" \
  --utility-port "$IB_UTILITY_PORT" \
  --license-type "$IB_LICENSE_TYPE" \
  --disable-telemetry true
```

The important design correction is that local all-in-one mode should enable Coordinator first and should not point the install at a fixed external Coordinator.

## What the new image should not do

The rebuilt image should not:

- Run an Initiator install against a fixed external Coordinator during Docker build.
- Assume a specific private IP address.
- Depend on Kubernetes to override a broken Docker command.
- Push directly to `latest` before validation.
- Commit raw recovered artifacts without review.
- Treat the Kubernetes example as the main user experience.

## Proposed runtime modes

### `IB_LAB_MODE=all-in-one`

Default mode.

Expected behavior:

1. Validate installer and required files exist.
2. Run local setup using a runtime-safe configuration.
3. Verify `/opt/incredibuild` and `/etc/incredibuild` exist.
4. Start Coordinator behavior.
5. Start server/agent behavior.
6. Start helper behavior.
7. Start UI/httpd behavior if supported.
8. Print status and useful endpoints.
9. Keep the container alive after startup.

### `IB_LAB_MODE=initiator`

Optional future mode.

Expected behavior:

1. Require `IB_COORDINATOR`.
2. Install or configure as an Initiator pointed at that Coordinator.
3. Use `--skip-coordinator-test true` only when intentionally requested for offline or staged setup.
4. Fail clearly if the Coordinator is not reachable and skip-test is not enabled.

This should not be the default mode.

## Proposed environment variables

```text
IB_LAB_MODE=all-in-one
IB_VERSION=4.13.0
IB_DATA_ROOT=/var/lib/incredibuild
IB_INSTALL_DIR=/opt/incredibuild
IB_COORDINATOR=localhost
IB_LOCAL_HTTP_PORT=8080
IB_LOCAL_HTTPS_PORT=8081
IB_UTILITY_PORT=9953
IB_LICENSE_TYPE=SUVM
IB_DISABLE_TELEMETRY=true
IB_ACCEPT_EULA=false
IB_MAX_INITIATOR_CORES=8
IB_RUN_SAMPLE=false
IB_KEEPALIVE=true
```

`IB_ACCEPT_EULA` should be explicit. The image should not silently accept license terms unless the user opts in.

## Proposed base image tooling

The rebuilt image should include a small set of diagnostic and build tools:

```text
bash
ca-certificates
curl
git
gcc
make
procps
iproute2
net-tools
```

`procps`, `iproute2`, and `net-tools` are included so basic commands such as `ps`, `ss`, and `netstat` are available during lab validation and troubleshooting.

## Proposed entrypoint behavior

Use an explicit `entrypoint.sh`.

High-level flow:

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Print startup banner and mode.
# 2. Validate installer or installed runtime path.
# 3. Require IB_ACCEPT_EULA=true before first-run install.
# 4. If not installed, perform first-run install based on IB_LAB_MODE.
# 5. Verify required binaries and config paths.
# 6. Start services in known order.
# 7. Optionally run sample build.
# 8. Print logs/status hints.
# 9. Keep foreground process alive.
```

## First-run install decision

The installer supports a local Coordinator role. This resolves the earlier open question.

The all-in-one model should be based on:

```text
--coordinator enabled
--initiator enabled
--helper enabled
```

not:

```text
--initiator enabled --coordinator-machine <old local IP>
```

The next validation step is service startup and listener validation after the completed first-run install.

## Service model finding

The extracted payload includes scripts such as:

```text
enable_coordinator.sh
enable_server.sh
enable_helper.sh
enable_httpd.sh
ib_service.bash
set_agent_params.py
manage_ui_access.sh
```

Inspection shows the enable/disable scripts are symlink-and-start wrappers. They rely on a completed install layout under `/opt/incredibuild` and `/etc/incredibuild`.

After first-run install, that layout now exists.

## Sample build path

The recovered Dockerfile tried to run:

```bash
git clone https://github.com/gilnadel/Sample.git
cd Sample
ib_console -f -p gcc.xml --no-cgroups make -j 100 ALL
```

The new project should not clone an external repo during default startup.

Better pattern:

- Use the bundled `samples/make_build` sample first.
- Include a documented optional sample build command.
- Gate sample execution behind `IB_RUN_SAMPLE=true`.

Validated sample docs from the payload indicate this shape:

```bash
ib_console make -j30
```

The exact path and command should be validated after service startup.

## Docker tag strategy

Do not move `latest` until a replacement image is validated.

Candidate tags:

```text
legacy-aio-20241110
aio-lab-YYYYMMDD
aio-lab-latest
```

## Kubernetes position

Kubernetes should remain optional.

The first successful path should be:

```text
docker run ...
```

Only after that works should the project add a Kubernetes example.

## Next engineering tasks

1. Start services after successful first-run install.
2. Confirm ports `8080`, `8081`, and `9953` listeners.
3. Confirm `ib_console`, `ib_info`, and service scripts work from the installed `/opt/incredibuild` path.
4. Validate the bundled `make_build` sample.
5. Draft `entrypoint.sh` skeleton.
6. Draft new `Dockerfile` skeleton using first-run install model.
7. Validate locally before publishing any image tag.
