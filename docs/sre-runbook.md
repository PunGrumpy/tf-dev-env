# tf-dev-env SRE runbook

Notes from building OpenSDN with tf-dev-env on a single Ubuntu 22.04 host
(16 vCPU, 30 GB RAM, 197 GB disk). Written from what was observed on that host;
items marked *(unverified)* were inferred from reading the scripts, not tested.

## 1. What runs where

| Component | Where | State |
|---|---|---|
| `run.sh` | host | stateless, regenerates `common.env`, `input/*.env` on every call |
| `tf-dev-sandbox` container | host Docker | **holds the build tree and stage markers** in its writable layer (~18.6 GB): `/root/work` (17 GB, of which `build/production` is 16 GB) and `/buildroot` (only 456 MB: the stripped install root that `docker build` copies from) |
| `tf-dev-env-registry` (`registry:2`, :5001) | host Docker | built images, in an anonymous Docker volume |
| `~/contrail` | bind mount to `/root/contrail` | sources from `repo sync` (5 GB) **and the real `.sconsign.dblite`** (~14 MB scons signature DB). `/root/work/.sconsign.dblite` is a 0-byte stub; scons replaces the symlink with a real file on write (see `freeze()` in `container/run.sh`) |
| `~/output` | bind mount to `/output` | unit test and container build logs |

Stage markers live in `/root/work/.stages` (`fetch`, `configure`, `compile`, `package`).
`run.sh build` skips stages that have a marker. A failed stage leaves no marker.

Two pieces of state live in different places and can drift apart: the scons signature DB is on
the host (`~/contrail`), while `build/` is inside the container. If the container is deleted, the
DB survives with entries for objects that no longer exist (observed: 277 stale `build/debug/*`
directories left over from a previous sandbox), so the first compile after a rebuild starts from
a mostly-empty `build/`.

## 2. Sizing (observed)

- Full build from a cold sandbox, measured end to end (09:21 to 11:04): ~16 min sandbox image,
  ~23 min `compile`, ~64 min `package` (rocky9 base, no layer cache), about 1 h 45 min in total.
  Unit tests: ~3.3 h for 37 targets.
- Disk peaks near 90 GB (Docker alone ~67 GB, plus a 27 GB `docker save` backup).
- `tf-dev-sandbox:compile` is a 21 GB image committed after `compile`.
- Install root vs build tree: `/buildroot` is 456 MB because binaries are stripped
  (`contrail-control`: 566 MB in `build/production`, 27 MB in `/buildroot`); the 16 GB
  `build/production` tree is what scons compares signatures against.
- Incremental compile works: editing one `.cc` in `controller/src/bfd` and rebuilding
  `libbfd.a` recompiled exactly one object in ~10 s. See section 7.
- Docker layer cache: rebuilding an unchanged leaf image by hand took 0.5 s (all layers `CACHED`)
  and produced the same image ID as the registry copy.

## 3. Do not

- `docker system prune -a`, `docker rm tf-dev-sandbox`, `docker rmi` of `tf-dev-sandbox:*`:
  deletes the build tree and stage markers; measured cost of recovering was ~1 h 45 min
  (see section 2). Take a `docker save` backup first (section 4).
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
`localhost:5001/*` tags back. Rehearsed once: the 26 GB tar was copied to a fresh Ubuntu 22.04
host with Docker 20.10.20 and `docker load` restored all 58 tags with 0 errors in under 20 min
(a tar saved by Docker 24.0.6 loads on 20.10). *(unverified: whether a container recreated from
the saved `tf-dev-sandbox:compile` image, a `docker commit` of the sandbox, brings back a usable
`/root/work` build tree together with the host's `.sconsign.dblite`)*

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
~1 h 45 min rebuild (measured). No source or log loss.

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

## 7. Incremental builds (measured on `controller/src/bfd`)

Requested through the same target every time, scons behaves as expected:

| Action | Result |
|---|---|
| edit one `.cc`, build `build/production/bfd/libbfd.a` | one object recompiled, ~10 s |
| edit only a comment | object is byte-identical, so `libbfd.a` is not rebuilt (scons cuts off on content) |
| build again with no change | nothing to do |

Caveats:

- The same object can read as stale or fresh depending on how it is requested. After building
  through `libbfd.a`, `scons -n build/production/bfd/bfd_session.o` reported it stale; requesting
  it through `libbfd.a` reported it fresh, and the two flip when you alternate. The mechanism is
  not understood. A stale-looking dry run on a directly requested `.o` is therefore not evidence of
  a broken cache.
- Always ask for the same target you actually use (a library, `install`, or the test target).
- Never run `scons` without `-n` in the sandbox just to look: it compiles and rewrites the signature DB.

## 8. Docker versions pinned by each tool

The tools that touch a host disagree about Docker, which is why they cannot share a host.

| Tool | Pins | Where |
|---|---|---|
| tf-dev-env | docker-ce 24.0.6 | `common/setup_docker_root.sh` |
| tf-devstack via kubespray (release-2.24) | docker-ce 20.10.20, containerd.io 1.6.16 | kubespray `docker` role defaults |
| tf-devstack via tf-ansible-deployer | docker-ce 28.5.2 (needs containerd.io >= 1.7.27) | `playbooks/roles/docker/tasks/Debian.yml` |

Other deploy-host notes from a working OpenStack + OpenSDN deployment (Ubuntu 22.04, no k8s):

- Do not add your own `download.docker.com` apt source next to the one tf-devstack creates
  (`Conflicting values set for option Signed-By`).
- A registry at `localhost:5001` is expected by `~/.tf/dev.env`; if the security group blocks port
  5001 between hosts, `scp` a `docker save` tar and run a local `registry:2` on the deploy host.
- Credentials are in `~/.tf/` (`passwords.yml`, `stack.env`) and
  `/etc/kolla/kolla-toolbox/admin-openrc.sh`; the default admin password is `contrail123`.
