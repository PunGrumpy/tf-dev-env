# tf-dev-env SRE runbook

Notes from building OpenSDN with tf-dev-env on a single Ubuntu 22.04 host
(16 vCPU, 30 GB RAM, 197 GB disk). Written from what was observed on that host;
items marked *(unverified)* were inferred from reading the scripts, not tested.

## 1. What runs where

| Component | Where | State |
|---|---|---|
| `run.sh` | host | stateless, regenerates `common.env`, `input/*.env` on every call |
| `tf-dev-sandbox` container | host Docker | **holds compile output and stage markers** (`/root/work`, `/buildroot`) in its writable layer |
| `tf-dev-env-registry` (`registry:2`, :5001) | host Docker | built images, in an anonymous Docker volume |
| `~/contrail` | bind mount to `/root/contrail` | sources from `repo sync` (5 GB) |
| `~/output` | bind mount to `/output` | unit test and container build logs |

Stage markers live in `/root/work/.stages` (`fetch`, `configure`, `compile`, `package`).
`run.sh build` skips stages that have a marker. A failed stage leaves no marker.

## 2. Sizing (observed)

- Full build from a cold sandbox: ~16 min sandbox image, ~23 min `compile`, ~60 min `package`
  (rocky9 base, no layer cache). Unit tests: ~3.5 h for 37 targets.
- Disk peaks near 90 GB (Docker alone ~67 GB, plus a 27 GB `docker save` backup).
- `tf-dev-sandbox:compile` is a 21 GB image committed after `compile`.

## 3. Do not

- `docker system prune -a`, `docker rm tf-dev-sandbox`, `docker rmi` of `tf-dev-sandbox:*`:
  deletes compile output; a rebuild takes hours.
- Run tf-devstack / kubespray on the same host. Its docker role reinstalls or removes Docker
  (see incident below).
- `run.sh` with no stage after local edits to `~/contrail`: `fetch` runs `repo sync` and can
  overwrite uncommitted changes.

## 4. Routine operations

Restart policy (set once; survives reboot):

    docker update --restart unless-stopped tf-dev-sandbox tf-dev-env-registry

Back up all images (about 26 GB, a few minutes):

    docker save $(docker images --format '{{.Repository}}:{{.Tag}}' \
      | grep -E '^localhost:5001/(opensdn|tf-dev-sandbox)|^tf-dev-sandbox:|^registry:2$' | sort -u) \
      -o ~/backup/images-$(date +%Y%m%d).tar
    sha256sum ~/backup/images-*.tar > ~/backup/SHA256SUM

Restore: `docker load -i ~/backup/images-<date>.tar`, start the registry, then push
`localhost:5001/*` tags back. *(a full restore onto a clean Docker has not been rehearsed)*

Rebuild only packaging after a failed `package` stage: `tf-dev-env/run.sh build`
(fetch/configure/compile are skipped because their markers exist).

## 5. Failure modes seen

| Symptom | Cause | Fix |
|---|---|---|
| `Version 5:24.0.6-1~ubuntu.20.04~focal not found` | Docker version pinned for focal on a jammy host | build the version string from `lsb_release` (`setup_docker_root.sh`) |
| `fatal: detected dubious ownership` in `schema_generate` (exit 128) | sources owned by host user, git runs as root | `git config --global safe.directory '*'` in the sandbox |
| `Unable to find a match: net-snmp-devel-1:5.9.1-17.el9` | Rocky repo only keeps the newest release | pin with a glob `5.9.1-*.el9` (in tf-container-builder) |
| `ERROR 403` fetching `downloadmirror.intel.com/...` | Intel removed the URL | harmless; image builds without the ice DDP firmware |
| `test-containers` fails (centos:7 repos, `opensdn-base:dev not found`) | centos:7 EOL, ordering race | not needed for deployment images |
| `agent` unit test `Metadata6RouteFuncsTest` | IPv6 disabled by cloud sysctl, vhost0 gets no link-local | enable IPv6 (`/etc/sysctl.d/99-zz-*.conf`) |
| `options_test` (collector, query-engine) | resolver search domain makes hostname `host.local` | keep `ip host` line in `/etc/hosts` in `run-tests.sh` |
| `"pyflakes" failed ... 'ExceptHandler' has no attribute 'depth'` | flake8 3.7.9 pinned in tox.ini, breaks on Python 3.9 | upstream `controller` repo change needed |
| opserver `KafkaClient ... no attribute '_closed'` | kafka-python version mismatch | not investigated |
| `ruleparser_dummy` reported as failure | placeholder test producing no XML | expected, ignore |

## 6. Incident: Docker removed by tf-devstack (2026-09-30)

**Impact:** all 57 built images, the sandbox container, the registry, and the compile cache were deleted;
~2 h rebuild needed. No source or log loss.

**Timeline**
1. tf-devstack (`k8s_manifests/run.sh`) ran kubespray on the build host. First attempt failed:
   kubespray wanted `docker-ce 20.10.20`, host had `24.0.6`; apt refused the downgrade.
2. Retried with `CONTAINER_RUNTIME=containerd`. kubespray's docker role ran
   *Reset | remove all containers*, *Remove docker package*, *Remove docker configuration files*.
   Docker, `/var/lib/docker`, and every container were gone. The host was also renamed `node1`,
   `/etc/hosts` and DNS search domains rewritten, swap disabled.
3. Recovery: `kubespray reset.yml`, restored hostname/hosts, `systemctl restart systemd-resolved`,
   reinstalled Docker via `run.sh`, re-enabled the service, rebuilt.

**Root causes**
- Build host and deploy tooling shared one Docker daemon; the deploy tool assumes it owns the node.
- The retry was made without reading what the containerd path does to an existing Docker install,
  and without a backup taken first.
- Compile output lived only in the container's writable layer.

**Actions**
- Take a `docker save` backup before any change to the build host (section 4).
- Keep deploy tooling off the build host.
- Set restart policies; consider bind-mounting `/root/work` so compile state survives container loss *(unverified)*.
