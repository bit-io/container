# container

Containers for the [bit.io](https://github.com/bit-io/bit) ecosystem, written in H#.

An OCI-compatible container library in the spirit of the `containers/*` libraries
(`image`, `storage`, `common`) plus a small native runtime. It does the same jobs: parse image
names, pull and push images, keep them in a local store, build images from a Containerfile,
generate OCI runtime specs and run the containers, with namespaces and cgroups.

```h#
use "bit -> container"

fn main() is
    let s = container::open_default_store()

    container::image_pull(s, "alpine:3.20")

    let mut o = container::container_options("alpine:3.20")
    o = container::set_option(o, "memory", "128m")
    o = container::set_option(o, "rm", "true")
    o = container::add_option(o, "volume", "/srv/data:/data:ro")
    o = container::set_command(o, ["/bin/echo", "hello from a container"])

    let code: int = container::container_run(s, o)
    write("exit code " + to_string(code))
end
```

Add it to a project with `bit install container`, or in `Bit.hk`:

```
[dependencies]
-> container => *
```

## Requirements

Linux, plus these tools (the library only uses standard util-linux / coreutils programs):

| required | `unshare nsenter pivot_root mount umount setpriv tar gzip curl sha256sum find sort comm cut mkfifo cp ps` |
|---|---|
| optional | `ip` + `iptables` (bridge networking), `setfattr` (overlay driver), `zstd` / `xz` / `bzip2` (compressed layers) |

Run `container::doctor()` (or `ctr doctor`) for a report of what this machine supports.
Most operations need **root**: mounts, `pivot_root`, cgroups and bridge networking are privileged.
Pulling, building without `RUN`, listing and the OCI layout import/export work without it.

## Concepts

| | |
|---|---|
| **Reference** | `alpine`, `user/app:1.2`, `ghcr.io/org/app@sha256:…`; parsed like Docker does. |
| **Store** | A directory (`/var/lib/container`, or `~/.local/share/container` for users) holding blobs, layers, images and containers. |
| **Driver** | `overlay` (layers stay separate, containers get an overlayfs mount) or `vfs` (each layer is a full copy; works everywhere). Chosen automatically, remembered in `store.json`. |
| **Options** | A flat `key=value` list describing a container. Single keys replace, `add_option` keys repeat. |
| **Container** | A directory `<store>/containers/<id>/` with `config.json` (OCI runtime spec), generated `init.sh` / `supervise.sh`, `console.log`, `pid`, `exit`. |

## API

All functions are flat: `container::<name>`. Types are named by module,
e.g. `container::store::Store`, `container::runtime::Options`; or simply let H# infer them
(`let s = container::open_default_store()`).

| area | functions |
|---|---|
| references | `parse_reference` `reference_canonical` `reference_familiar` |
| digests | `digest_of_file` `digest_of_string` `digest_valid` |
| store | `open_store(root, run_root, driver)` `open_default_store` `default_store_root` |
| images | `image_pull` `image_push` `image_list` `image_resolve` `image_exists` `image_inspect` `image_config` `image_tag` `image_remove` `image_import` `image_import_with` `image_export` `image_load` `image_gc` `image_new_config` |
| registries | `registry_login` `registry_logout` |
| build | `build_image(store, context, containerfile, tag, build_args, net)` |
| options | `container_options` `set_option` `add_option` `set_command` `option_get` `option_flag` |
| containers | `container_create` `container_start(attach)` `container_run` `container_stop` `container_kill` `container_wait` `container_exec` `container_remove` `container_list` `container_find` `container_status` `container_exit_code` `container_pid` `container_logs` `container_logs_tail` `container_inspect` `container_bundle` |
| host | `host_info` `doctor` `version` |

Container options:

| key | meaning |
|---|---|
| `name`, `hostname`, `workdir`, `user` | `user` may be a name (resolved through the image's `/etc/passwd`) or `uid[:gid]` |
| `arg` (repeat) | command + arguments (replaces the image CMD); `entrypoint` (repeat) replaces ENTRYPOINT |
| `env` (repeat) | `KEY=value` |
| `volume` (repeat) | `/host/path:/in/container[:ro]`, or `name:/in/container` for a named volume in `<store>/volumes/` |
| `network` | `host`, `none`, or `bridge` (default `bridge` when root and `ip` exist, else `host`) |
| `publish` (repeat) | `8080:80`, `127.0.0.1:8080:80/udp` (bridge only) |
| `memory`, `cpus`, `pids`, `shares` | cgroup limits: `256m`, `1.5`, `100`, `512` |
| `readonly`, `privileged`, `userns` | read-only root, keep all capabilities, add a user namespace |
| `rm`, `detach` | for `container_run` |
| `cap_add`, `cap_drop` (repeat) | capability names; `ALL` is understood |
| `rootfs_dir` | run directly in this directory instead of an image (used by the builder) |

## Containerfile builder

`FROM` (image or `scratch`), `ARG`, `ENV`, `WORKDIR`, `USER`, `LABEL`, `EXPOSE`, `VOLUME`,
`ENTRYPOINT`, `CMD`, `STOPSIGNAL`, `COPY`, `ADD` (local files and tar archives), `RUN`.
`RUN` executes in a real container on the build root; after every `RUN` / `COPY` / `ADD`
the changes become a gzip layer, deletions are recorded as `.wh.` whiteouts.

Not supported: multi-stage builds and `COPY --from`, `ADD` of URLs, `.dockerignore`, build cache,
`HEALTHCHECK` / `SHELL` / `ONBUILD` (ignored).

## How the native runtime isolates a container

`unshare` creates mount, UTS, IPC, PID (and network / user) namespaces. A generated init script
assembles the root filesystem (overlay mount or private copy), mounts `/proc`, `/sys`, a tmpfs
`/dev` with the standard device nodes, applies volumes, masks sensitive `/proc` paths, then
`pivot_root`s into it and detaches the old root. PID 1 of the container is the command itself,
started through `setpriv` with the configured uid/gid, a reduced bounding set (Docker's default
capability set unless changed) and `no_new_privs`. A supervisor records the pid and exit code.
Limits go into cgroup v2 (`cpu.max`, `memory.max`, `pids.max`) or the v1 equivalents.

Every container directory is also an **OCI bundle** (`config.json` + `rootfs/` with the `vfs`
driver), so `container::container_bundle(s, id)` can be handed to `runc run -b` / `crun run -b`.

## Limits you should know about

* It is a small runtime built from standard tools, **not a hardened sandbox**. There is no seccomp
  profile, no AppArmor / SELinux handling and no user-namespace remapping by default. Do not use it
  to run hostile code; run `runc` / `crun` on the generated bundle if you need that.
* `exec` joins the container's namespaces and cgroup. Capabilities are dropped only when the image
  contains `setpriv`; otherwise a warning is printed and the command keeps the host's capabilities.
* No TTY / PTY allocation: interactive shells work when the calling terminal is inherited
  (`attach=true`), but there is no `-t`.
* Rootless mode (`user` namespace with `--map-root-user`) exists but has no overlay, cgroups or
  bridge networking, and has seen little testing.
* The store has no locking; do not run two writers on one store.
* Layer extraction relies on GNU `tar`; image layers are trusted not to contain hostile symlink tricks.
* Registry support covers Bearer and Basic auth (no credential helpers, no `docker-archive` import).

## Testing

```
bit check          # type-check
bit test           # runs the #[test] functions in src/test_*.h#
```

The tests create real containers and therefore need root; on hosts without root or util-linux
those tests return early. The registry tests start `tests/mock_registry.py` (python3) on a random
loopback port and are skipped without it. They use a minimal rootfs assembled from the host's own
shell and libraries, so no network access is needed.

## Example

`examples/ctr` is a docker-like command line built on the library:

```
cd examples/ctr && bit build
./ctr pull alpine:3.20
./ctr run --rm -m 64m alpine:3.20 /bin/sh -c 'echo hi; hostname'
./ctr build -t demo:1 ../myapp
```

## License

Apache-2.0
