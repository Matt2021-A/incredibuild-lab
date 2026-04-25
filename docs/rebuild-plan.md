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
3. Enable or start Coordinator behavior if supported.
4. Enable or start server/agent behavior if supported.
5. Enable or start helper behavior if supported.
6. Enable UI/httpd behavior if supported.
7. Print status and useful endpoints.
8. Keep the container alive after startup.

### `IB_LAB_MODE=initiator`

Optional future mode.

Expected behavior:

1. Require `IB_COORDINATOR`.
2. Install or configure as an Initiator pointed at that Coordinator.
3. Fail clearly if the Coordinator is not reachable.

This should not be the default mode.

## Proposed environment variables

```text
IB_LAB_MODE=all-in-one
IB_VERSION=4.13.0
IB_DATA_DIR=/etc/incredibuild
IB_INSTALL_DIR=/opt/incredibuild
IB_COORDINATOR=localhost
IB_MAX_INITIATOR_CORES=8
IB_RUN_SAMPLE=false
IB_KEEPALIVE=true
```

These are placeholders until the service model is confirmed.

## Proposed entrypoint behavior

Use an explicit `entrypoint.sh`.

High-level flow:

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Print startup banner and mode.
# 2. Validate installer or installed runtime path.
# 3. If not installed, perform first-run install based on IB_LAB_MODE.
# 4. Verify required binaries and config paths.
# 5. Start services in known order.
# 6. Optionally run sample build.
# 7. Print logs/status hints.
# 8. Keep foreground process alive.
```

## First-run install open question

The critical technical question is how to install or activate a local Coordinator without requiring an external Coordinator.

The old build ran:

```text
--initiator enabled --coordinator-machine <old local IP> --license-type SUVM
```

That is the wrong default for the rebuilt lab image.

The next inspection pass needs to determine whether the installer supports a Coordinator install mode, an all-in-one role combination, or post-install toggles using the management scripts.

## Service model open question

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

These likely modify service configuration, database state, symlinks, or init behavior. They must be inspected before writing the real entrypoint.

## Sample build path

The recovered Dockerfile tried to run:

```bash
git clone https://github.com/gilnadel/Sample.git
cd Sample
ib_console -f -p gcc.xml --no-cgroups make -j 100 ALL
```

The new project should not clone an external repo during default startup.

Better pattern:

- Include a documented optional sample build command.
- Prefer a small local sample checked into the repo if licensing permits.
- Gate sample execution behind `IB_RUN_SAMPLE=true`.

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

1. Inspect the service scripts inside the extracted payload.
2. Inspect the samples directory.
3. Determine installer flags for local Coordinator or all-in-one setup.
4. Draft `entrypoint.sh` skeleton.
5. Draft new `Dockerfile` skeleton using first-run install model.
6. Validate locally before publishing any image tag.
