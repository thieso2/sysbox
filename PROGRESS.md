# Sync Fork with nestybox/sysbox:master — COMPLETED

## Goal
Sync thieso2/sysbox with nestybox/sysbox:master (v0.7.0) and validate all fork-specific fixes are preserved.

## What Was Done

### 1. Synced master branch
- Fast-forwarded `master` from `694d4d8` to `27cb713` (upstream v0.7.0)
- 8 upstream commits incorporated (submodule updates, docs, v0.7.0 release)

### 2. Rebased all submodules with fork changes
- **sysbox-runc**: 6 fork commits (--run-dir flag) rebased onto upstream `a4dd414`
- **sysbox-pkgr**: 1 fork commit (AppArmor fusermount3) rebased onto upstream `b8403a9`
- **sysbox-libs**: fast-forwarded to upstream `9b42cd4` (no fork changes)
- **sysbox-fs**, **sysbox-mgr**, **sysbox-ipc**: unchanged upstream base, fork changes preserved
- **sysbox-dockerfiles**: fork changes preserved

### 3. Fixed go.mod staleness
- `sysbox-fs` and `sysbox-mgr` needed `go mod tidy` after sysbox-libs update
- Committed updated go.mod/go.sum to both repos

### 4. Forked missing repos
- Created `thieso2/dockerfiles` and `thieso2/sysbox-libs` forks on GitHub
- Simplified CI workflow: removed URL override hacks (all repos now forked)

### 5. Rebuilt dockyard branch
Clean 5-commit history on top of upstream v0.7.0:
```
3753a51 Update sysbox-fs and sysbox-mgr: go mod tidy for Go 1.24
d95b5a9 CI: remove submodule URL overrides (all repos now forked)
1068211 Bump VERSION to 0.7.0.1-tc
b87bd95 Add --run-dir flag and all fork-specific submodule changes
3324d75 Add static build CI workflow and CLAUDE.md
```

## Validation Results

### CI Build (GitHub Actions) — ALL GREEN
- x86_64 static build: PASS (1m34s)
- aarch64 static build: PASS (1m41s)

### Incus VM Testing (Ubuntu 24.04 on 100.106.185.92)
- All three binaries built and installed successfully
- Version reported: `0.7.0.1-tc`
- `--run-dir` flag present and accepted in all three binaries (sysbox-runc, sysbox-mgr, sysbox-fs)
- sysbox-mgr and sysbox-fs start and create sockets in `/run/sysbox/`
- AppArmor drop-in installed at `/etc/apparmor.d/local/sysbox-fs-fusermount`
- ID-mapped mounts detected as supported

### Known Issue (upstream, not fork-related)
Container execution fails with `unsafe procfs detected: openat2 fsmount` error.
This is an **upstream sysbox v0.7.0 issue** — confirmed by testing with pure upstream
sysbox-runc binary (same error). Likely related to sysbox-runc's new "safe procfs"
security hardening interacting with the KVM/Incus VM environment. Does not affect
bare-metal installations.

## Steps
- [x] Identify upstream changes since fork point
- [x] Catalog all fork-specific commits
- [x] Create GitHub issue (#9)
- [x] Perform the sync (rebase submodules + rebuild dockyard)
- [x] Validate all fixes are present after sync
- [x] Run CI to confirm builds pass
- [x] Test on Incus VM
- [x] Clean up test VM

## Status: COMPLETE — [#9](https://github.com/thieso2/sysbox/issues/9)
