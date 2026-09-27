# WASIX lower-layer ownership review

| | | Remarks |
| --- | --- | --- |
| **Status** | ▶️ | Architecture findings captured for later implementation. |
| **Severity** | Medium | The existing compatibility code works, but several fixes are owned by lower layers and should benefit every WASIX guest. |

## Context

This review compares the local `libuv-wasix` changes with the current Wasmer
WASIX and `wasix-libc` implementations. The baseline inspected on 2026-08-06
was:

- `libuv-wasix` submodule at `71cdbb575`, with an uncommitted `src/unix/fs.c`
  change;
- Wasmer `sdk` at `4a84ca03f18`;
- the current `wasix-org/wasix-libc` `main` source.

The goal is not to eliminate every WASIX branch in libuv. It is to keep each
behavior in the lowest layer that can implement it truthfully and generally:

```text
libuv        owns uv_* to POSIX adaptation
wasix-libc   owns POSIX to WASIX translation
Wasmer       owns descriptor, process, and socket behavior
```

libuv should retain platform adaptation that is intrinsic to its API. It should
not reconstruct WASIX descriptor metadata, emulate process notifications, or
report successful socket configuration that the runtime did not apply.

## Decision Matrix

| Current libuv-wasix behavior | Preferred owner | Direction |
| --- | --- | --- |
| Convert a zero access mode to `O_RDONLY` | `wasix-libc` | Move now |
| Retry directory opens with `O_DIRECTORY` | None after the libc fix | Remove if regression tests confirm it is redundant |
| Repair `st_mode` using `__wasi_fd_fdstat_get()` | Wasmer plus `wasix-libc` | Move now |
| Poll child processes every 100 ms | Wasmer WASIX process signaling | Replace with event-driven child notification |
| Return success for unsupported multicast options | `wasix-libc` and Wasmer networking | Move now |
| Disconnect UDP by clearing only libuv state | Wasmer networking plus `wasix-libc` | Replace later with a real operation |
| Read byte-only IPC streams without `recvmsg()` | libuv temporarily | Keep until the lower stack supports `recvmsg()` |
| Adapt `void(void *)` thread entries to `pthread_create()` | libuv | Keep and harden |
| Translate `uv_process_options_t` through `posix_spawn()` | libuv | Keep |
| Enable libuv's `SO_REUSEPORT` path for WASIX | libuv | Keep |

## Filesystem Findings

### Zero access mode

The local `src/unix/fs.c` wrapper turns an access mode of zero into
`O_RDONLY`. This compensates for `wasix-libc` defining `O_RDONLY` as an
explicit bit and rejecting `oflag & O_ACCMODE == 0` in its `openat()`
translation. Code such as Node's `uvwasi` may pass a literal zero because that
is the conventional POSIX read-only representation.

The general fix belongs in `wasix-libc`:

```c
if ((oflag & O_ACCMODE) == 0)
  oflag |= O_RDONLY;
```

Apply this before the access-mode switch in:

```text
wasix-libc/libc-bottom-half/cloudlibc/src/libc/fcntl/openat.c
```

This makes `open(path, 0)`, `open(path, O_CLOEXEC)`, and equivalent callers
read-only for every WASIX C guest. A libuv-only normalization would leave the
same compatibility failure in other libraries.

### Directory retry

The same libuv wrapper retries a failed directory open with `O_DIRECTORY`.
Current Wasmer `path_open2` already permits an existing `Kind::Dir` without
that flag; `O_DIRECTORY` only enforces that the target must be a directory.
The observed `EINVAL` may therefore be entirely caused by the missing access
mode in `wasix-libc`.

After fixing the zero-mode translation, validate `open(directory, 0)` and
`open(directory, O_RDONLY)`. Delete the directory retry if both work. Do not
retain a `stat()`-then-`open()` retry without a demonstrated second failure: it
adds a race and duplicates runtime path resolution.

### Descriptor type consistency

The local libuv patch calls `__wasi_fd_fdstat_get()` when `fstat()` returns no
POSIX file type. This is a layering smell and exists because Wasmer maintains
two sources of descriptor metadata:

- `fd_filestat_get` returns the cached `InodeVal::stat`;
- `WasiFs::fdstat` derives a file type independently from `Kind` and contains
  special handling for descriptors 0, 1, and 2.

Wasmer should have one internal descriptor-kind mapping used when an inode is
created and whenever `Fdstat` or `Filestat` is produced. The mapping should
distinguish at least:

```text
regular file
directory
symbolic link
terminal character device
unidirectional pipe
socket pair / duplex pipe
TCP, UDP, raw, and sequential-packet sockets
```

For exact POSIX pipe behavior, add a WASIX `Filetype::Pipe` extension and map it
to `S_IFIFO` in `wasix-libc`. `Kind::PipeRx` and `Kind::PipeTx` should report
that type, while `Kind::DuplexPipe` created by `socketpair()` should remain
`SocketStream`. Existing consumers that do not recognize the added WASIX value
will continue treating it as unknown, which is no worse than the current pipe
metadata.

Redirecting a pipe onto fd 0, 1, or 2 must preserve the pipe's type. Descriptor
numbers alone must not imply `CharacterDevice`; terminal status should come
from the actual descriptor or virtual file. The prototype on Wasmer's
`origin/isatty` branch is useful prior art, but the final invariant should be
shared descriptor metadata rather than a separate special case.

Once `fd_filestat_get()` and `fd_fdstat_get()` agree, remove the raw WASI
fallback and `<wasi/api.h>` dependency from libuv's filesystem code.

## Child Process Notification

The WASIX libuv loop currently caps an otherwise infinite poll at 100 ms while
process handles exist, then calls `uv__wait_children()`. This avoids a hang but
adds idle wakeups and up to 100 ms of child-exit latency.

Wasmer already tracks child completion and implements blocking and nonblocking
`proc_join`. The missing behavior is notifying the parent in a form that wakes
its event loop. The preferred implementation is:

1. Populate `WasiProcess::parent` when fork/spawn creates a child.
2. Publish the child main thread's final status before notification.
3. Queue `Signal::Sigchld` on the parent when the child finishes.
4. Ensure signal delivery wakes a parent blocked in a WASIX poll/deep sleep.
5. Let libuv's normal signal-driven reaping call `waitpid(WNOHANG)`.
6. Remove `UV__WASIX_CHILD_POLL_MS` and the unconditional post-poll reap.

If libuv's signal watcher cannot be made reliable on WASIX, the fallback design
should be an explicitly pollable process-completion handle, not periodic
polling embedded in every libuv event loop.

The WASIX-specific `posix_spawn()` translation remains valid libuv code.
Wasmer cannot interpret `uv_process_options_t`, libuv stdio containers, or uv
process flags.

## Networking Findings

### Multicast options

The current libuv WASIX branches return success for multicast TTL, loopback,
and interface selection without necessarily changing the socket. Wasmer's
`VirtualUdpSocket` already exposes IPv4/IPv6 multicast loopback and IPv4
multicast TTL operations. The missing bridge is primarily the POSIX option
translation in `wasix-libc`.

Map standard options such as `IP_MULTICAST_TTL`, `IP_MULTICAST_LOOP`, and their
IPv6 equivalents onto the existing WASIX socket-option calls. Add a runtime
operation for multicast interface selection if the current interface cannot
represent it. Until it is implemented, return a stable unsupported error
rather than false success.

After this mapping works through `setsockopt()`, remove the corresponding
libuv no-op branches and use its ordinary Unix implementation.

### UDP disconnect

The current WASIX branch of `uv__udp_disconnect()` only clears
`UV_HANDLE_UDP_CONNECTED`. The underlying Wasmer socket can remain connected,
so later operations may still use the previous peer. This is a semantic
workaround, not a complete disconnect.

The correct design is a `VirtualUdpSocket::disconnect()` operation exposed by
WASIX and called by `wasix-libc` when `connect()` receives `AF_UNSPEC`. Host
networking can use the host disconnect operation; proxy/browser networking can
clear its logical peer. This crosses the virtual networking interfaces and is
therefore a second-stage task, but it should ultimately replace the libuv-only
state change.

### `SO_REUSEPORT`

Keep the libuv platform gate that enables `SO_REUSEPORT` on WASIX. Wasmer and
`wasix-libc` already support the operation; the remaining issue is that libuv
otherwise refuses to call it. This is a legitimate libuv platform adaptation.

## IPC and Threading Findings

### Byte-only IPC

Keep the current plain `read()` path for libuv IPC streams for now. WASIX does
not yet provide `msghdr`/`SCM_RIGHTS` descriptor passing, while byte-only Node
IPC remains useful. A lower-layer `recvmsg()`/`sendmsg()` implementation that
supports iovecs and rejects control data could later remove this branch; full
handle passing requires coordinated ABI, libc, and Wasmer work.

### Thread trampoline

Keep the libuv thread-entry trampoline. The old implementation calls a
`void(void *)` function through a `void *(void *)` function pointer. Native
ABIs often tolerate that undefined behavior, but WebAssembly indirect calls
require an exact function signature. Wasmer should not weaken typed indirect
calls to accommodate it.

Harden the trampoline before considering it complete:

- copy the entry function and argument, then free the allocation before
  invoking the entry, so `pthread_exit()` cannot leak it;
- free the allocation when `pthread_create()` returns an error.

## Proposed Work Packages

### Package A: remove the local libuv filesystem workaround

1. Add `wasix-libc` tests for zero-mode file and directory opens.
2. Normalize zero access mode to `O_RDONLY` in `wasix-libc`.
3. Add Wasmer consistency tests comparing `fdstat` and `filestat` for files,
   directories, socketpairs, pipes, and redirected stdio.
4. Centralize Wasmer descriptor-kind classification.
5. Add pipe-to-`S_IFIFO` support across WASIX types and `wasix-libc` if exact
   pipe classification is still required.
6. Delete the uncommitted `libuv-wasix/src/unix/fs.c` compatibility code.
7. Re-run Node `node:wasi`, Rolldown, child-process stdio, and TTY tests.

### Package B: remove false multicast success

1. Add direct C tests for multicast TTL and loopback through `setsockopt()`.
2. Map the options in `wasix-libc` to existing WASIX operations.
3. Add any missing virtual-network interface selection operation.
4. Delete libuv's multicast no-op branches.
5. Re-run Node `dgram` and reuse-port tests.

### Package C: event-driven child completion

1. Wire parent ownership into `WasiProcess`.
2. Deliver and wake on `SIGCHLD` after child status publication.
3. Test a child exit while the parent has no timers or I/O.
4. Remove the 100 ms libuv poll workaround.
5. Re-run `spawn`, `exec`, `fork`, cluster, and IPC lifecycle tests.

### Package D: deferred socket and IPC completeness

1. Add real UDP disconnect across virtual networking and `wasix-libc`.
2. Add byte-only `sendmsg()`/`recvmsg()` support below libuv.
3. Design descriptor transfer separately; do not conflate it with byte-only
   IPC compatibility.

## Validation Gates

Lower-layer replacements are complete only when:

- the relevant focused C/WASIX regression passes without libuv-specific code;
- the existing EdgeJS Node compatibility test still passes;
- native libuv behavior is unchanged;
- browser/JS and sys Wasmer backends use the same WASIX-facing operation;
- removed shims are deleted rather than left as alternate paths.

## Conclusion

The uncommitted libuv filesystem patch should not be committed in its current
form. Zero-mode opening belongs in `wasix-libc`, and descriptor type truth
belongs in Wasmer with POSIX conversion in `wasix-libc`. Multicast no-ops and
periodic child polling should also migrate downward. The thread trampoline,
`posix_spawn()` adapter, `SO_REUSEPORT` platform enablement, and temporary
byte-only IPC path remain appropriate libuv responsibilities.
