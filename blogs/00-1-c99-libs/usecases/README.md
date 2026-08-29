# USE CASES — 12 docker-compose stacks (2 per library) · L10

Each of the six libraries, two **completely different** use cases, as self-contained compose stacks that
**download the released `.so`** (auto-selecting amd64/arm64), verify it, and inject one real fault into a
real target. Every stack has a four-part write-up below: *what it does · how it works · why nothing else
can · how we do it.*

---

## Intro — how to download and utilize the libraries

**The artifacts.** Every tagged release ships **24 prebuilt objects** plus a checksum manifest:

```
libchaos-<lib>-<libc>-<arch>.so        # lib ∈ {io,net,dns,time,memory,process}
                                       # libc ∈ {glibc,musl}   arch ∈ {amd64,arm64}
SHA256SUMS
```
Download URL (semver tag, **no `v`** prefix):
```
https://github.com/macstab/macstab-chaos-testing-libraries/releases/download/1.0.0/libchaos-io-glibc-amd64.so
```

**Pick the right one — two axes, both non-negotiable:**
- **arch:** `amd64` on Intel, `arm64` on Apple Silicon / Graviton (`uname -m` → `x86_64|aarch64`).
- **libc:** `glibc` for Debian/Ubuntu/RHEL images, `musl` for Alpine. **You cannot load a `-glibc-` object
  in an Alpine container** — glibc and musl are separate ABIs. (All target images below are Debian-bookworm,
  so these stacks fetch `glibc`.)

**Best-practice wiring — the `fetch` sidecar (used by every stack here):** a throwaway `alpine` init service
downloads the arch-correct `.so` + `SHA256SUMS` into a shared volume, **verifies the checksum**, copies it to
a stable name, and exits 0; the app container then preloads it. Zero modification to the app image.

```yaml
fetch:
  image: alpine:3.20
  environment: { CHAOS_VERSION: ${CHAOS_VERSION:-1.0.0} }
  volumes: [ "chaoslibs:/libs" ]
  command:
    - sh
    - -ec
    - |
      apk add --no-cache wget ca-certificates >/dev/null
      ARCH=$$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/')
      B="https://github.com/macstab/macstab-chaos-testing-libraries/releases/download/$$CHAOS_VERSION"
      cd /libs
      wget -q "$$B/libchaos-io-glibc-$$ARCH.so" "$$B/SHA256SUMS"
      sha256sum -c SHA256SUMS --ignore-missing 2>&1 | grep "libchaos-io-glibc-$$ARCH.so: OK"   # fail-closed
      cp "libchaos-io-glibc-$$ARCH.so" libchaos-io.so
```

**Then, to utilize it, three rules that never change:**
1. **Scope the preload** — set `LD_PRELOAD` on the one process, via `env … exec`, and **never `export`** it
   (it would poison `git`/`ssh`/`cron`). Compose several libs space-separated: `LD_PRELOAD="…/io.so …/net.so"`.
2. **The config is the control plane** — a plain text file at the **hardcoded** path `/tmp/.chaos-<lib>.conf`
   (no env override). Mount a **writable** `/tmp` (read-only rootfs blocks it). Ramp it live with `sed`; `rm`
   it to disable — fail-open on the next syscall.
3. **Mind `AT_SECURE`** — a setuid privilege drop (`gosu`) makes the kernel strip `LD_PRELOAD`. Drop privilege
   with **`setpriv`** (Debian) or **`su-exec`** (Alpine) — capability-based, no setuid bit — or the interceptor
   silently never loads.

**Prerequisites & run.** A Linux Docker host (the containers are Linux; the library is Linux-only). Then:
```bash
export CHAOS_VERSION=1.0.0        # override to pin a different release
docker compose -f usecases/mem-1-enomem.yml up --abort-on-container-exit
docker compose -f usecases/mem-1-enomem.yml down -v
```

**The errno-slot rule (why configs differ per library):** io/net/dns put the **concrete errno in the effect
slot** (`connect:ECONNREFUSED:1.0`); time/memory/process use the **literal `ERRNO` token** with the errno in
the value (`execve:ERRNO:EACCES`); special effects use their own token (`TORN CORRUPT OFFSET FAIL_AFTER
REWRITE FILTER_FAMILY LIMIT SHUFFLE TIMEOUT LATENCY`). net has 4 fields (`endpoint:op:effect:value`).

| Library | Stack | Fault | Config line |
|---|---|---|---|
| io | `io-1-postgres-fsync` | WAL `fdatasync`→`EIO`→PANIC | `…/pg_wal:fdatasync:EIO:0.05` |
| io | `io-2-torn-corrupt` | bit-flip on read | `/data:read:CORRUPT:0.5` |
| net | `net-1-redis-refused` | `connect`→refused (Redis up) | `tcp4://*:6379:connect:ECONNREFUSED:1.0` |
| net | `net-2-http-timeout` | `recv`→timeout | `tcp4://*:80:recv:ETIMEDOUT:1.0` |
| dns | `dns-1-rewrite-canary` | reroute to canary | `dns://primary:REWRITE:canary` |
| dns | `dns-2-eai-again` | intermittent resolve failure | `dns://backend:EAI_AGAIN:0.5` |
| time | `time-1-jwt-expiry` | realtime +1h → JWT expired | `clock_gettime/realtime:OFFSET:3600000` |
| time | `time-2-ratelimiter` | monotonic steps backward | `clock_gettime/monotonic:OFFSET:-5000` |
| memory | `mem-1-enomem` | anon `mmap`→`ENOMEM` | `mmap/anon:ERRNO:ENOMEM` |
| memory | `mem-2-latency` | `mmap` +150 ms | `mmap:LATENCY:150` |
| process | `proc-1-thread-exhaustion` | 8 threads then `EAGAIN` | `pthread_create:FAIL_AFTER:EAGAIN,8` |
| process | `proc-2-subprocess-blocked` | `execve`→`EACCES`, parent lives | `execve:ERRNO:EACCES` (+posix_spawn) |

---

## §io-1 — Postgres WAL `fdatasync` returns `EIO`

- **What it does.** Fails 5% of Postgres WAL `fdatasync()` calls under a live write load. Postgres treats a
  WAL sync error as unrecoverable: it raises a **PANIC** and enters crash recovery, taking the data directory
  down — while the app may still be accepting connections. The exact production incident coarse tools miss.
- **How it works.** `libchaos-io` interposes `fdatasync()`; it resolves the fd to its path via
  `readlink(/proc/self/fd/N)`, matches it longest-prefix / path-boundary-aware against
  `/var/lib/postgresql/data/pg_wal`, and on a probability hit sets `errno=EIO` and returns `-1` **before** the
  real syscall.
- **Why nothing else can.** `dm-flakey` needs a dedicated device-mapper device and root; Toxiproxy/iptables
  live at the network layer and can't touch a filesystem syscall; nothing else gives you **per-path,
  per-syscall `EIO` at a probability you flip live from a text file**.
- **How we do it.** `/var/lib/postgresql/data/pg_wal:fdatasync:EIO:0.05` (io grammar: `path:op:errno:prob`).
  Note `fdatasync ≠ fsync` — Postgres uses `fdatasync` for WAL, so we target it specifically. Privilege is
  dropped with `setpriv`, not the setuid `gosu`, so `AT_SECURE` doesn't strip the preload.

## §io-2 — silent bit-flip on read (`CORRUPT`)

- **What it does.** A process writes a 4 KB record, then reads it back 20× verifying a SHA-256. Half the reads
  come back with a **single flipped bit** → the app reports `CHECKSUM MISMATCH`. Corruption, not failure.
- **How it works.** `libchaos-io` interposes `read()`/`pread()`; after the real read succeeds it flips one bit
  in the returned buffer (post-call mutation) at the configured probability, scoped to `/data`.
- **Why nothing else can.** This lives in the **data plane** — the bytes the app already trusts. A
  connection-level proxy or a pod-killer is a layer removed from the buffer; only an interposer sitting on
  `read()` can corrupt what the application just received. It exercises the checksum/journaling recovery paths
  that are otherwise never fired.
- **How we do it.** `/data:read:CORRUPT:0.5`. `CORRUPT` is `read`/`pread`-only — you can't flip a bit you
  never see, so `O_DIRECT`/`sendfile`/`splice` (kernel-to-kernel) are out of scope, honestly.

## §net-1 — client `connect()` refused while Redis is healthy

- **What it does.** Redis is up the entire time; the **client's** `connect()` to 6379 is refused 100%. Tests
  the client's retry / circuit-breaker behaviour against a demonstrably working backend.
- **How it works.** `libchaos-net` interposes `connect()`; it normalizes the peer `sockaddr` to an endpoint,
  matches `tcp4://*:6379`, and returns `-1`/`ECONNREFUSED` before the syscall.
- **Why nothing else can.** `iptables`/proxies fail the whole connection or topology and add a TCP hop worth
  ~50–200 µs; only syscall interposition fails a **specific `connect()` per-endpoint** at ~150–300 ns, with no
  `NET_ADMIN`, while the server stays provably alive.
- **How we do it.** `tcp4://*:6379:connect:ECONNREFUSED:1.0` — net's 4-field grammar
  (`endpoint:op:effect:value`); `*` = any IP on that port.

## §net-2 — HTTP `recv()` times out mid-response

- **What it does.** nginx serves fine; the client's `connect` and `send` succeed, but `recv()` returns
  `ETIMEDOUT` mid-response — the failure that strands connection pools and exposes missing read timeouts.
- **How it works.** `libchaos-net` interposes `recv()`/`recvmsg()`; on a matched, already-established session
  it returns `ETIMEDOUT` instead of delivering bytes.
- **Why nothing else can.** This is an **in-session** failure: `iptables` sees packets, not `recv()` calls; a
  proxy would reset the connection, not time out one read. Only interposition can fail a single `recv()` while
  the socket stays open.
- **How we do it.** `tcp4://*:80:recv:ETIMEDOUT:1.0`.

## §dns-1 — reroute a hostname to a canary (`REWRITE`)

- **What it does.** The client asks for `primary` and reaches the **canary** backend — silently rerouted, no
  client change, no `/etc/hosts` edit.
- **How it works.** `libchaos-dns` interposes `getaddrinfo()`; on selector `dns://primary` it rewrites the
  queried name to `canary` **before resolution**, so the canary's address is returned.
- **Why nothing else can.** `dnsmasq`/`hosts` answer at the protocol layer *after* a round trip and can't
  rewrite a **successful** answer; a proxy sits below name resolution. Only interposition at `getaddrinfo`
  performs answer-set surgery before a packet leaves — and it sits *above* NSS, so `/etc/hosts` still works.
- **How we do it.** `dns://primary:REWRITE:canary` (dns's 3-field grammar: `selector:effect:value`).

## §dns-2 — intermittent `EAI_AGAIN`

- **What it does.** The backend is healthy; 50% of `getaddrinfo` lookups return `EAI_AGAIN` (transient). Tests
  whether the client retries with backoff — the failure mode that cascades into service unavailability when a
  nameserver hiccups.
- **How it works.** `libchaos-dns` interposes `getaddrinfo()`; on a probability hit it returns `EAI_AGAIN`
  before any resolver work happens.
- **Why nothing else can.** You can't make a real resolver emit `EAI_AGAIN` on demand without a round trip;
  `hosts`/`dnsmasq` give you NXDOMAIN or valid answers. Interposition injects the transient code **hermetically
  — no DNS server in the loop.**
- **How we do it.** `dns://backend:EAI_AGAIN:0.5`.

## §time-1 — `CLOCK_REALTIME` +1h makes a fresh JWT expire

- **What it does.** A token minted with the **real** clock (valid 5 minutes) reads as **EXPIRED** when verified
  under a +1 h realtime offset. Stdlib-only HS256, no network.
- **How it works.** `libchaos-time` interposes `clock_gettime()`; for `CLOCK_REALTIME` it adds +3,600,000 ms to
  the returned `timespec`, so `time.time()` in the verifier is an hour ahead → `exp < now`. The token is minted
  in a non-preloaded step (real `exp`) and verified in a preloaded one.
- **Why nothing else can — the moat.** `System.currentTimeMillis()`/`nanoTime()` are `@IntrinsicCandidate`:
  the JIT inlines them to a CPU register read (`RDTSC`/`CNTVCT_EL0`), so an **in-JVM agent has no call site to
  hook**. And `clock_gettime` is a **vDSO** read, not a syscall — so `eBPF`/`seccomp` don't see it either. Only
  libc-level interposition sits *under* the intrinsic.
- **How we do it.** `clock_gettime/realtime:OFFSET:3600000` (signed ms; `OFFSET` is `clock_gettime`-only).

## §time-2 — `CLOCK_MONOTONIC` steps backward

- **What it does.** Two consecutive monotonic reads with the offset changed between them → the second is 5 s
  earlier than the first → **elapsed goes negative**, corrupting every timeout / backoff / rate-limiter that
  depends on it.
- **How it works.** `OFFSET` is applied per call from the *current* config; rewriting the config file between
  the two reads makes `CLOCK_MONOTONIC` step backward — exactly what a VM migration or an NTP step does.
- **Why nothing else can.** Monotonic is the one clock nothing is supposed to move backward. You can't do it
  from inside the runtime (intrinsic) and you can't do it with a proxy. Only a **per-call** libc offset can
  violate the invariant deterministically.
- **How we do it.** Start `clock_gettime/monotonic:OFFSET:0`, then rewrite to `:OFFSET:-5000` between reads.
  (A *constant* offset can't break a delta — the mid-flight jump is the whole point.)

## §mem-1 — anonymous `mmap` returns `ENOMEM`

- **What it does.** A 4 MB anonymous `mmap` fails with `ENOMEM` while file-backed mappings are untouched — the
  OOM a JVM `-Xmx` can never trigger (direct buffers / thread stacks / heap growth live in the anonymous arena).
- **How it works.** `libchaos-memory` interposes `mmap()`; it matches on the **`MAP_ANONYMOUS` bit**
  (`mmap/anon`) and returns `MAP_FAILED`/`ENOMEM` before the syscall.
- **Why nothing else can.** Actually exhausting RAM invokes the **OOM killer** — non-deterministic, and it
  SIGKILLs your process (the wrong test). A `-Xmx` limit governs a different arena. Only **call-shape**
  interposition fails exactly the anonymous arena, deterministically, with no killer.
- **How we do it.** `mmap/anon:ERRNO:ENOMEM` (memory grammar: literal `ERRNO` token; `mmap/anon` vs
  `mmap/file` selector). Note `M_MMAP_THRESHOLD` is adaptive (128 KB → up to 32 MB), so this hits the arena, not
  a fixed size.

## §mem-2 — `mmap` latency injection

- **What it does.** Every `mmap` gains 150 ms → allocation-path / page-reclaim **tail latency** with no error.
- **How it works.** `libchaos-memory` interposes `mmap()` and sleeps the configured latency before the real
  call. Timing in the probe is *real* — only the memory lib is preloaded, so `clock_gettime` is untouched.
- **Why nothing else can.** You can't add deterministic latency to the allocation syscall from userspace
  tooling; cgroup memory pressure is coarse and non-deterministic. Interposition adds **exactly 150 ms to
  exactly `mmap`.**
- **How we do it.** `mmap:LATENCY:150`.

## §proc-1 — thread-pool exhaustion (`FAIL_AFTER`)

- **What it does.** The first 8 `pthread_create()` calls succeed; the 9th onward return `EAGAIN` — a
  deterministic "we ran out of threads under load."
- **How it works.** `libchaos-process` interposes `pthread_create()`; a per-operation counter allows N, then
  returns `EAGAIN` — **returned directly**, because the pthreads ABI returns the errno rather than setting the
  global `errno`. The counter resets on config reload.
- **Why nothing else can.** "Succeed N times, then fail" is a **stateful** fault probabilistic tools can't
  express; and the real ceiling (`RLIMIT_NPROC` / the `pids` cgroup / `kernel.threads-max`) is non-deterministic
  to reach in a test box.
- **How we do it.** `pthread_create:FAIL_AFTER:EAGAIN,8` (process grammar: `FAIL_AFTER:<errno>,<N>`).

## §proc-2 — subprocess exec blocked (`EACCES`), parent survives

- **What it does.** A subprocess exec is blocked with `EACCES` while the **parent keeps running** — the exact
  scope that reveals whether your shell-out / `ProcessBuilder` / `subprocess` code treats a failed spawn as
  fatal.
- **How it works.** `libchaos-process` interposes `execve()`/`execveat()` **and** `posix_spawn()`/
  `posix_spawnp()` — CPython 3.12's `subprocess` may take either path — and fails the exec on a hit; the child
  never runs, the parent receives the error.
- **Why nothing else can.** `seccomp` can block `execve` but **not without also blocking the `fork`/`clone`** —
  it can't express "the exec fails but the process survives." Only interposition fails exactly the exec.
- **How we do it.** three lines: `execve:ERRNO:EACCES`, `posix_spawn:ERRNO:EACCES`, `posix_spawnp:ERRNO:EACCES`.

---

## Honesty (not spin)
- These are **validated for compose schema and DSL grammar** (`docker compose config` passes on all 12; every
  config line matches the per-library grammar). They are **not** execution-verified — run them on a Linux Docker
  host to confirm the fault fires.
- Known risk points: `io-1` needs `setpriv` present in `postgres:16-bookworm` (util-linux — expected); `time-2`
  depends on the config reload landing between two reads (increase the jump if your host coalesces them);
  `proc-2` hooks both exec paths but a truly exotic spawn path would need its own line; and the fetch step
  assumes the `1.0.0` release assets exist with the documented names — override with `CHAOS_VERSION`.
