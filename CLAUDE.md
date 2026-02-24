# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## GitHub Workflow

- Create issues and PRs in the fork **thieso2/sysbox**, not the upstream nestybox/sysbox.
- Working branch: `dockyard`

## Build Commands

Builds run inside a privileged Docker container to avoid polluting the host. The host needs Docker, Make, and Git.

```bash
make sysbox            # normal build (inside container)
make sysbox-static     # static linking (inside container)
make sysbox-debug      # with debug symbols (inside container)

# Cross-compile:
make sysbox TARGET_ARCH=arm64

# Install/uninstall (requires root):
sudo make install      # copies binaries to /usr/bin/
sudo make uninstall
```

Build artifacts land in `<component>/build/<arch>/` (e.g., `sysbox-runc/build/amd64/sysbox-runc`).

**Local builds** (skip the Docker container, used inside the test/build container):
```bash
make sysbox-local         # depends on: sysbox-ipc, then sysbox-runc, sysbox-fs, sysbox-mgr
make sysbox-static-local  # static variants
```

Build dependency order: `sysbox-ipc` (generates .pb.go) must build first, then the three binaries can build in parallel.

## Testing

All tests run inside privileged Docker containers. Host kernel must be >= 5.12.

```bash
make test                # all test suites
make test-sysbox         # integration tests (bats framework)
make test-runc           # sysbox-runc unit tests
make test-fs             # sysbox-fs unit tests
make test-mgr            # sysbox-mgr unit tests
make test-sysbox-libs    # sysbox-libs unit tests

# Run specific integration test:
make test-sysbox TESTPATH=tests/sysfs
make test-sysbox TESTPATH=tests/sysfs/disable_ipv6.bats

# Interactive debug shell inside the test container:
make test-shell

# Inside that shell, run a test directly:
bats -t tests/sysfs/disable_ipv6.bats

# Cleanup test directories (requires root):
sudo make test-cleanup
```

```bash
make lint    # go vet + go fmt on all components, shellcheck on tests
make shfmt   # format shell scripts
```

## Architecture

Sysbox is a container runtime (OCI runc fork) composed of three cooperating daemons:

- **sysbox-runc** (`sysbox-runc/`) — OCI runtime, spawned per container by Docker/containerd. Handles container lifecycle (create/start/exec/delete). Makes gRPC calls to the other two daemons.
- **sysbox-fs** (`sysbox-fs/`, entry: `cmd/sysbox-fs/main.go`) — Long-running daemon. FUSE-based /proc and /sys virtualization + seccomp syscall trapping. Gives containers a VM-like filesystem view.
- **sysbox-mgr** (`sysbox-mgr/`, entry: `main.go`) — Long-running daemon. Manages uid/gid sub-ranges, rootfs cloning/chowning, mount coordination, shiftfs/idmapped-mount decisions.

### IPC (sysbox-ipc)

All inter-daemon communication is **gRPC over Unix domain sockets** defined in `sysbox-ipc/`:

| Socket | Server | Client | Proto |
|--------|--------|--------|-------|
| `/run/sysbox/sysmgr.sock` | sysbox-mgr | sysbox-runc | `sysboxMgrGrpc/sysboxMgrProtobuf/` |
| `/run/sysbox/sysfs.sock` | sysbox-fs | sysbox-runc | `sysboxFsGrpc/sysboxFsProtobuf/` |
| `/run/sysbox/sysfs-seccomp.sock` | sysbox-fs | sysbox-runc | seccomp tracer |

The `.pb.go` files are checked in; you only need `protoc` if you change the `.proto` files.

sysbox-runc's gRPC client wrappers live in `sysbox-runc/libsysbox/sysbox/` (`mgr.go`, `fs.go`).

**Container lifecycle flow:** sysbox-runc create → Register with sysbox-mgr → Register with sysbox-fs → start → Update both → delete → Unregister both.

### Shared Libraries (sysbox-libs)

`sysbox-libs/` contains independent Go modules (each with own `go.mod`): idMap, shiftfs, overlayUtils, mount, pidmonitor, capability, linuxUtils, dockerUtils, containerdUtils, etc.

### Submodules

All major components are Git submodules (see `.gitmodules`): sysbox-fs, sysbox-runc, sysbox-ipc, sysbox-mgr, sysbox-libs, sysbox-pkgr, sysbox-dockerfiles. Uses relative URLs (`../sysbox-fs.git`). Clone with `--recursive`.

### Helper Scripts

- `scr/sysbox` — starts sysbox-fs + sysbox-mgr daemons (`sudo ./scr/sysbox [--debug]`)
- `scr/docker-cfg` — configures Docker to use sysbox-runc as a runtime
- Logs: `/var/log/sysbox-fs.log`, `/var/log/sysbox-mgr.log`

### Key Build Tags

- `seccomp`, `apparmor` — used by sysbox-runc
- `idmapped_mnt` — enabled on kernels >= 5.12 (sysbox-runc, sysbox-mgr)
- `netgo`, `osusergo` — used for static builds (all components)
