# Upstream Contribution Plan for nestybox/sysbox

## Contribution Requirements (from CONTRIBUTING.md)

- File an issue first for each non-trivial change
- Branch naming: `<issue-number>-description`
- Commit messages: imperative mood, max 70 char summary, `Signed-off-by:` (DCO)
- Run `gofmt -s -w` on all Go files
- Tests required (Go `testing` + `bats` integration)
- Docs changes in same commit as code
- Squash into logical commits before merge
- Reference issues with `Fixes #XXX`

---

## Proposed Patch Series (3 patches, in dependency order)

### Patch 1: AppArmor fusermount3 drop-in

**Upstream issue to file:** Reference existing [#947](https://github.com/nestybox/sysbox/issues/947) (already open, maintainer ctalledo diagnosed the same problem)

**Repos:** `nestybox/sysbox-pkgr` + `nestybox/sysbox`

**Rationale:**
On Ubuntu 24.04+, AppArmor ships a restrictive `fusermount3` profile that
blocks FUSE mount operations except at explicitly allowed paths. sysbox-fs
uses FUSE to mount under `/var/lib/sysboxfs/`, which is denied by default.
This causes all sysbox containers to fail to start on affected systems.

The standard fix is to ship a `local/` drop-in (the AppArmor-endorsed
extension mechanism) rather than modifying the upstream profile. The maintainer
already proposed the same rules in #947 but no PR was filed to automate the
install.

**Changes:**

*sysbox-pkgr:*
- New file: `systemd/sysbox-apparmor-fusermount` — the drop-in policy
- `deb/sysbox-ce/rules` — install the drop-in during deb build
- `deb/sysbox-ce/sysbox-ce.postinst` — append include line to `/etc/apparmor.d/local/fusermount3`, reload profile
- `deb/sysbox-ce/sysbox-ce.postrm` — remove include line on purge, reload profile

*sysbox (top-level):*
- `Makefile` — install/uninstall the drop-in in `make install`/`make uninstall` (for non-deb installs)

**Why this is patch 1:** No dependencies on other patches. Already has upstream buy-in via #947. Smallest, most self-contained change. High urgency (breaks all Ubuntu 24.04+ users).

---

### Patch 2: Fix IDMappedMount volume ownership for non-ID-map-capable filesystems

**Upstream issue to file:** New issue. Related to existing #906, #812, #754.

**Repo:** `nestybox/sysbox-mgr`

**Rationale:**
When `rootfsUidShiftType` is `IDMappedMount`, sysbox-mgr creates backing
directories for container volumes (e.g., `/var/lib/sysbox/docker/<id>/...`)
owned by `0:0`, assuming the kernel's ID-mapped mount will translate ownership.

However, if the backing filesystem (e.g., a volume mount, NFS, or certain
overlay configurations) does not support ID-mapped mounts, sysbox-runc
silently falls back to a regular bind mount. The directory remains owned by
real root (UID 0), but inside the container the process runs as the mapped
UID (e.g., 165536) — making the directory inaccessible. This breaks
Docker-in-Docker because inner dockerd cannot write to `/var/lib/docker`.

The fix: when IDMappedMount is the shift type, create directories owned by
`uidMappings[0].HostID:gidMappings[0].HostID` instead of `0:0`. This works
correctly in both cases:
- With ID-mapped mounts: the kernel translates the offset back to UID 0 inside the container
- Without ID-mapped mounts: the container's root UID directly owns the directory

**Change (2-line fix in `mgr.go`):**
```diff
 case idShiftUtils.IDMappedMount:
     volChownOnSync = false
-    volUid = 0
-    volGid = 0
+    volUid = info.uidMappings[0].HostID
+    volGid = info.gidMappings[0].HostID
```

**Why this is patch 2:** Self-contained 2-line fix in a single file. No dependencies. Fixes a real bug affecting users on filesystems without ID-mapped mount support.

---

### Patch 3: Add `--run-dir` flag for multi-instance support

**Upstream issue to file:** New issue (enhancement).

**Repos:** `nestybox/sysbox-ipc`, `nestybox/sysbox-runc`, `nestybox/sysbox-mgr`, `nestybox/sysbox-fs`, `nestybox/sysbox`

**Rationale:**
All sysbox runtime paths are hardcoded to `/run/sysbox/` (sockets, pid files)
and `/var/lib/sysboxfs/` (FUSE mountpoint). This makes it impossible to run
multiple independent sysbox instances on the same host — a requirement for:

- **Multi-tenant isolation**: each tenant gets a dedicated sysbox instance
  with its own socket namespace, preventing cross-tenant interference
- **Testing/CI**: run integration tests against a dev build without
  interfering with the production sysbox instance
- **Distrobox/toolbox**: run sysbox in unprivileged contexts with
  user-specific runtime directories

The `--run-dir <dir>` flag (default: `/run/sysbox`) configures the directory
for all runtime files (sockets, pid files). It is accepted by all three
binaries and the `scr/sysbox` wrapper script.

**Changes (in dependency order within the patch):**

1. *sysbox-ipc* — Change socket address constants to variables, add `SetSockAddr()`:
   - `sysboxFsGrpc/grpcServer.go`: `const → var`, add `SetSockAddr()`
   - `sysboxMgrGrpc/grpcServer.go`: same

2. *sysbox-runc* — Add `--run-dir` CLI flag with env var propagation:
   - `main.go`: add `--run-dir` to CLI flags
   - `libsysbox/sysbox/sysbox.go`: `init()` parses `os.Args` for `--run-dir`
     (workaround for urfave/cli v1 limitation with subcommands), falls back to
     `SYSBOX_RUN_DIR` env var, calls `SetRunDir()` which updates socket
     addresses and exports the env var for re-exec'd children

3. *sysbox-mgr* — Add `--run-dir` CLI flag:
   - `main.go`: add flag, update `sysboxRunDir`, pid file path, and gRPC socket

4. *sysbox-fs* — Add `--run-dir` CLI flag:
   - `cmd/sysbox-fs/main.go`: add flag, update run dir, pid file, gRPC socket,
     and seccomp tracer socket; change `const → var` for mutability
   - `seccomp/tracer.go`: `const → var`, add `SetSeccompTracerSockAddr()`

5. *sysbox (top-level)* — Update wrapper script:
   - `scr/sysbox`: accept `--run-dir`, pass to mgr/fs, export `SYSBOX_RUN_DIR`

**Design decisions:**
- Env var `SYSBOX_RUN_DIR` for sysbox-runc because it is not a long-running
  daemon — it's invoked per-container by the container runtime, which doesn't
  pass custom flags. The env var is set by `scr/sysbox` and inherited by all
  sysbox-runc invocations.
- `os.Args` parsing in `init()` is necessary because urfave/cli v1 does not
  reliably make global flags available in `app.Before` when a subcommand is
  present (e.g., `sysbox-runc --run-dir /x create`). The flag must be resolved
  before any gRPC connections are attempted.
- Default remains `/run/sysbox` — zero behavior change for existing users.

**Why this is patch 3:** Largest change, touches 5 repos. Novel feature with no upstream precedent. Benefits from the other patches landing first (especially the AppArmor patch, since `--run-dir` would affect FUSE mount paths in a future follow-up). May require more discussion with maintainers.

---

## Submission Order

```
Patch 1 (AppArmor)  →  lands first, most urgent, upstream buy-in exists
Patch 2 (IDMapped)  →  lands second, small bug fix, self-contained
Patch 3 (--run-dir) →  lands last, feature, needs discussion
```

## Pre-Submission Checklist

For each patch:
- [ ] File upstream issue (or reference existing #947 for patch 1)
- [ ] Create branch named `<issue>-description` in each affected repo
- [ ] Run `gofmt -s -w` on all changed `.go` files
- [ ] Add `Signed-off-by:` to all commits
- [ ] Squash into single logical commit per repo
- [ ] Update docs if applicable
- [ ] Run `make test` (requires Linux host with kernel >= 5.12)
- [ ] Open PRs in all affected repos, cross-referencing each other
