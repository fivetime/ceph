# `Containerfile.fivetime` — ceph v20.2.4 with three fixes that are in no release

This is an **overlay** on `quay.io/ceph/ceph:v20.2.4`, not a source build. That
is not a shortcut: it is the shape the three fixes actually have. Two of them
are mgr Python files that the official image ships verbatim, and the third is
not a ceph change at all.

## The three

| # | Defect | Upstream | What it costs us | Carried how |
|---|---|---|---|---|
| 1 | `mgr/rook` joins every address of a k8s Node with `/` into `HostSpec.addr`, so `ceph nfs cluster info` cannot resolve it | [ceph/ceph#69250](https://github.com/ceph/ceph/pull/69250), **still open** | Manila's `_get_export_ips()` calls exactly that — **the Manila NFS integration is blocked on this one defect** | cherry-pick `7ac32114d88`, 2 production files |
| 2 | `mgr/prometheus` calls `node_proxy_fullreport()` every scrape; the rook orchestrator does not implement it and mgr records a crash dump per call | [#70967](https://github.com/ceph/ceph/pull/70967) → tentacle backport [#71041](https://github.com/ceph/ceph/pull/71041), merged 2026-08-25 — **seven days after v20.2.4 was tagged** | 109 crash dumps; HEALTH_ERR when archiving raced with new arrivals | cherry-pick `88a54adccf4`, 1 file — **Python half only**, see below |
| 3 | ganesha answers `OP_GET_DIR_DELEGATION` (opcode 46) with `OP_ILLEGAL`; a kernel-7.0 NFS client turns that into `EREMOTEIO` | fixed in nfs-ganesha **V9.11**; the image carries **V5.9** | ~13% of `ls` calls exit non-zero on 7.0 clients (tenant 6.8 nodes are clean) | upgrade the CentOS Storage SIG package |

### What #2 does *not* include

The tentacle backport is three commits. Only `88a54adccf4` is Python. The other
two — `03272001c80` and `3ddab990f00` — change `src/mgr/ActivePyModules.cc` and
would need a full ceph build.

They are the belt to the Python braces: they stop mgr recording a crash **if**
the call happens. The Python guard stops the call happening **at all**, which is
the half that removes our symptom. If the C++ half is ever needed, this overlay
is the wrong tool — build from `container/Containerfile`.

### Why the ganesha fix is a package, not a patch

Upstream fixed this a long time ago. We are on V5.9 only because that is what
the CentOS Storage SIG had built for el9s when the image was made, and `ceph`
has no version pin for it anywhere watchable — it follows the base image's
repos. So there is nothing to patch: install a newer build of the same SIG
package. `centos-release-nfs-ganesha<N>` brings both the repo definition and the
SIG signing key, which the ceph image does not otherwise carry.

`GANESHA_SIG_STREAM=9` is the default because 9.16 is the smallest jump from the
shipped 5.9 that carries the fix. `15` is the newest series and resolves just as
cleanly. Neither pulls or replaces a single ceph package — measured, not assumed:

```
Upgrading: libntirpc  nfs-ganesha  nfs-ganesha-ceph  nfs-ganesha-rados-grace
           nfs-ganesha-rados-urls  nfs-ganesha-rgw  nfs-ganesha-selinux
```

## Building

```bash
git checkout v20.2.4-fivetime
docker build -f container/Containerfile.fivetime -t ceph:local .
```

The build context is three files, not the ceph tree: `Containerfile.fivetime.dockerignore`
cuts it from roughly a gigabyte to 60 kB. Requires BuildKit, which honours a
per-Dockerfile ignore file.

The build **fails** rather than producing an image that only looks patched: it
greps for the removed slash-join, for `InternalIP`, for the node-proxy guard,
byte-compiles all three files, and asserts `nfs-ganesha >= GANESHA_MIN_VERSION`.

## CI

`.github/workflows/build-fivetime-image.yml`, on push to `v20.2.4-fivetime` or
by dispatch. It builds with `--load`, **verifies the finished image, and only
then pushes** — a cache-served layer whose input changed is exactly what a
post-push check would miss.

Two tags per build:

- `ghcr.io/fivetime/ceph:v20.2.4-fivetime` — moving, for humans
- `ghcr.io/fivetime/ceph:v20.2.4-fivetime-<sha>` — pinned

**Use the pinned tag in `CephCluster.spec.cephVersion.image`.** A tag that moves
under Rook is an unplanned restart of every mon, mgr, OSD, MDS and NFS pod.

Dispatch has a guard: `workflow_dispatch` requires the workflow file on the
default branch, which means it can also be dispatched *with* the default branch
selected. `main` is ceph 21.0.0; copying its mgr Python onto a 20.2.4 image
would produce something that starts and misbehaves. The job compares the source
tree's `project(... VERSION)` against the base image tag and refuses a mismatch.

## Deploying

Changing `CephCluster.spec.cephVersion.image` makes Rook roll **the whole
cluster**. Do it deliberately, and check `ceph -s` between stages.

Afterwards the three checks that matter:

```bash
ceph nfs cluster info <name>                    # 1: must resolve, not "Cannot resolve IP for host"
ceph crash ls | tail                            # 2: no new node_proxy_fullreport entries
kubectl -n rook-ceph exec <nfs pod> -c nfs-ganesha -- ganesha.nfsd -v   # 3: >= V9.11
```

For 3, the discriminating check is not the version string but the client
behaviour: a kernel-7.0 client's `ls` in a loop should stop returning non-zero.
