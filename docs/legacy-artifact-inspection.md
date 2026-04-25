# Legacy Artifact Inspection

## Purpose

This document captures the findings from the recovered Dockerfile, Kubernetes manifest, and deployment automation. The goal is to preserve the useful intent of the original project without carrying forward broken startup behavior or old environment assumptions.

## Files inspected

Recovered files reviewed from the original project archive:

```text
incredibuilddockerfile
incredibuild-deployment.yaml
DeployIncredibuildKubernetes.ps1
Install-Prerequisites.ps1
Kubernetes Info.txt
```

Raw recovered files should not be committed without review. The deployment script and notes contain private/local environment details and old credentials. Treat those values as burned and keep them out of the public repo.

## Dockerfile findings

The recovered Dockerfile is named `incredibuilddockerfile`, not `Dockerfile`.

It uses this historical base:

```dockerfile
FROM iwashuman2021/incredibuild:20241110
```

It installs basic build tooling:

```dockerfile
RUN apt-get update
RUN apt-get install -y gcc make git
```

It sets Incredibuild-related values:

```dockerfile
ENV IB_VERSION=4.13.0
ENV IB_COORDINATOR=192.168.2.243
EXPOSE 8080
EXPOSE 8081
```

It downloads the Incredibuild 4.13.0 installer and runs an Initiator install against a fixed Coordinator address:

```dockerfile
ADD https://ib-downloads-official.s3.amazonaws.com/incredibuild_4.13.0.run ./incredibuild_4.13.0.run
RUN chmod a+x ./incredibuild*
RUN ./incredibuild_4.13.0.run --action install --initiator enabled --coordinator-machine 192.168.2.243 --data-dir /etc/ --license-type SUVM
```

That matches the install-log finding from the public `latest` image: the install attempted to use the same hard-coded Coordinator address and failed when that Coordinator was unreachable on port `9953`.

## Dockerfile command problem

The Dockerfile contains two `CMD` instructions.

The first command block attempted to start the Incredibuild agent, clone a sample repo, and run a sample build:

```dockerfile
CMD bash -c -x " \
  /opt/incredibuild/management/set_agent_params.py max-initiator-cores 8; \
  /opt/incredibuild/etc/init.d/incredibuild start; sleep 10; \
  git clone https://github.com/gilnadel/Sample.git; \
  cd Sample; \
  ib_console -f -p gcc.xml --no-cgroups make -j 100 ALL; \
"
```

The second command immediately overwrote that startup behavior:

```dockerfile
CMD ["incredibuild"]
```

Docker only honors the last `CMD`, so the useful startup block was not the effective runtime command. This explains why the public image now fails at startup with:

```text
exec: "incredibuild": executable file not found in $PATH
```

## Dockerfile interpretation

The recovered Dockerfile was not truly all-in-one in its final form. It was closer to an Initiator image that depended on an external Coordinator at build time and then lost its intended runtime startup block because of the final `CMD` instruction.

The all-in-one intent still matters, but the implementation did not preserve that intent cleanly.

## Kubernetes manifest findings

The recovered static manifest defines:

- Namespace: `incredibuild`
- Deployment: `ib-coordinator`
- Deployment: `ib-initiator`
- Services:
  - `ib-coordinator-ui` on port `8000`
  - `ib-coordinator-server` on port `31100`
  - `ib-license` on port `50052`
  - `ib-initiator-comm` on port `31104`

The static manifest has empty `image:` values. The PowerShell script generates a more complete manifest using the Docker Hub image tag.

Both deployments set `IB_COORDINATOR`, but neither deployment defines `command:` or `args:`.

## Kubernetes command override finding

The Kubernetes deployment did not override the broken Docker `CMD`.

Because the manifest does not set `command:` or `args:`, Kubernetes would use the image default command. For the public image, that default command is:

```json
["incredibuild"]
```

Since there is no `incredibuild` executable in `$PATH`, both Kubernetes pods would inherit the same startup failure unless another runtime mutation happened outside the recovered manifest.

## Kubernetes architecture mismatch

The manifest names one deployment `ib-coordinator` and another `ib-initiator`, but both use the same image. The recovered Dockerfile installs as an Initiator against an external Coordinator. It does not build a local Coordinator image.

That means the `ib-coordinator` deployment name is misleading. It exposes Coordinator-like ports, but the image itself was not clearly built or configured as a Coordinator.

## Deployment script findings

`DeployIncredibuildKubernetes.ps1` automates:

1. Docker Hub login
2. Dockerfile generation if missing
3. Pulling `iwashuman2021/incredibuild:20241110`
4. Building `iwashuman2021/incredibuild:latest`
5. Pushing `latest` to Docker Hub
6. Setting a Kubernetes context
7. Creating the `incredibuild` namespace
8. Generating and applying Kubernetes YAML

Important issues:

- It contains local Windows paths.
- It contains old private credentials and must not be committed raw.
- It bakes in the same local Coordinator address used by the Dockerfile.
- It builds and pushes directly to `latest`.
- It does not solve the broken image `CMD` problem.
- It does not generate Kubernetes `command:` or `args:` overrides.

## Prerequisite script findings

`Install-Prerequisites.ps1` is a general workstation bootstrap script. It checks or installs:

- Chocolatey
- AWS CLI
- Azure CLI
- Docker Desktop
- kubectl
- Helm

This is useful context for the original project but should not be part of the minimal rebuilt lab path. The new project should not require a Windows workstation bootstrap before a user can understand the container.

## Current conclusion

The recovered artifacts show useful intent, but the implementation mixed three models:

1. A Docker image intended to run a sample build.
2. An Initiator install pointed at an external Coordinator.
3. A Kubernetes deployment that names Coordinator and Initiator pods but does not actually configure different startup behavior.

That mix explains the current broken state.

## Recovery direction

The rebuilt project should choose one clear model:

```text
First-run local lab install with explicit startup orchestration.
```

The rebuild should not install against a fixed external Coordinator during image build. It should not rely on Kubernetes to compensate for a broken container command. It should not push directly to `latest` before validation.

## Follow-up inspection still needed

The content of these scripts still needs to be captured from inside the `latest` image before finalizing the entrypoint design:

```text
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/ib_service.bash
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/enable_coordinator.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/enable_server.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/enable_helper.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/enable_httpd.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/disable_coordinator.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/disable_server.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/disable_helper.sh
/opt/incredibuild_tmp/incredibuild_tmp-x86_64/management/disable_httpd.sh
```

Capture summaries, not raw logs, unless the content is clearly safe and useful.
