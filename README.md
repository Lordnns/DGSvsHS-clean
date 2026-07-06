# DGSvsHS — Scenario-Dependent Energy Efficiency of ECS Dedicated Game Servers

Reproducible benchmark harness for the paper **"Scenario-Dependent Energy Efficiency of ECS
Dedicated Game Servers"**.

The study holds one cooperative survivor-shooter workload fixed and reproduces it **bit-for-bit
across three ECS dedicated-server runtimes**, varying only the server engine and measuring
server-side CPU utilization (a proxy for power at fixed clock) as the enemy population scales
geometrically from ~1,700 to ~35,000 entities over ten rounds. All three servers speak a **single
versioned QUIC wire protocol** and are driven by two independent client engines (Unity and Unreal)
using a deterministic autopilot, so the client is removed as a confound and the server runtime is
isolated as the variable under test.

**Central finding:** the energy-optimal server engine is *scenario-dependent*. Rust/Bevy is most
efficient below ~25k entities, Bevy and Unity DOTS reach parity near 25k, Unity DOTS leads by ~35k
(per-body vs per-island physics sleeping), and Unreal MASS — forced to marshal every enemy to and
from the non-ECS Chaos solver each tick — saturates a core early and is never competitive.

---

## The three server legs

| Leg | Runtime | ECS | Physics | ECS-native physics | Transport | Port | Directory |
|---|---|---|---|---|---|---|---|
| **Unity DGS** | C# / Unity 6 Dedicated Server | DOTS / Entities | DOTS Physics | ✅ | QUIC (quinn `GameSocket`) | **7777** | `DGSvsHS/` |
| **Unreal DGS** | C++ / UE 5.7 headless | MASS | Chaos (marshalled each tick) | ❌ | QUIC (MsQuic) | **7780** | `Unreal/UnrealvsHS/` |
| **Bevy** | Rust / static-musl | Bevy ECS | avian2d | ✅ | QUIC (quinn) | **4433** | `rust/` |

Only the **server** is benchmarked. Clients run on a separate host so client load never contaminates
the measurement. The workload runs a fixed **62.5 Hz (16 ms) tick**; enemy population per round is
`N(r) = round(1700 · 1.4^(r-1))` for `r ∈ [1,10]` → `1700, 2380, 3332, 4665, 6531, 9143, 12800,
17920, 25088, 35124`.

### Client ↔ server connectivity (paper Table II)

Each native-engine server is bound to its own engine's QUIC socket library, so a client can only
drive a server it shares a socket implementation with. Bevy's quinn socket is reachable from both.

|              | Unity DGS | Unreal DGS | Bevy |
|--------------|:---------:|:----------:|:----:|
| Unity client |     ✅     |     —      |  ✅  |
| Unreal client|     —     |     ✅      |  ✅  |

> **Legacy note — the "Arch" leg.** An earlier iteration had a fourth leg: a plain-.NET **C# Arch
> ECS + BepuPhysics** server on port **7778**. It was superseded by the Unreal leg and its server
> directory is no longer in the repo, but vestigial hooks remain: the Unity `BuildModeSwitcher` still
> offers an `HS/Arch` target and `test_harness/run_case.py` still accepts `--flavor arch`. **The Arch
> leg is not part of this paper** — ignore those hooks unless you are resurrecting that server.

---

## Repo layout

```
DGSvsHS/                     Unity 6 project — Unity client + Unity DOTS DGS server
  Assets/_Game/
    Editor/BuildModeSwitcher.cs   One-click define/port switcher (DGS | HS/Bevy | BareBone)
    Net/Quic/                     QUIC client + server (DllImports the native cdylib)
    Net/WireFormat.md             Authoritative wire-protocol spec (all legs implement this)
    Server/DedicatedServerMain.cs Unity DGS entrypoint (QuicNetworkServer, port 7777)
    Server/Dots/                  DOTS/Entities ECS simulation
    Harness/TrialRunner.cs        Client-side autopilot + per-tick NDJSON logger
  build_microvm_{aarch64,x86_64}.sh   Unity DGS → Linux microVM / ISO

Unreal/UnrealvsHS/           UE 5.7 project — Unreal client + Unreal MASS DGS server
  Source/UnrealvsHS/
    Client/   UvHSClientGameMode.cpp, UvHSAutoPilot.cpp   (client + autopilot, --bot-id CLI)
    Server/   SimRunner/SimSystems/SimContext              (headless MASS sim)
    Mass/     UvHSMassTypes.h                               (MASS fragments)
    Net/      QuicServer/QuicClient/WireCodec (MsQuic)
  Source/ThirdParty/MsQuic/  Bundled MsQuic headers/libs
  build_microvm_{aarch64,x86_64}.sh   Unreal DGS → Linux microVM / ISO
  build_microvm_x86_64_trace.sh       Same, with tracing instrumentation

rust/                        Rust/Bevy headless server workspace
  cli/                       Server entrypoint binary (parses --seed/--duration/--god-mode)
  gameplay/                  Bevy ECS sim + avian2d physics + QUIC plugin
  stress_client/             Standalone Rust load generator
  build_microvm_{aarch64,x86_64}.sh   Bevy → Linux microVM / ISO (--god-mode / --tracy flags)

native/quic_client/          Rust QUIC cdylib (dgsvshs_socket) — Unity loads it via DllImport
  scripts/build_and_deploy.{ps1,sh}   Build + copy into Unity Assets/Plugins/

test_harness/                Python benchmark orchestration + analysis
  run_case.py                Drives a full N-run case (recorder + clients + cooldown)
  record_run.py              20 Hz Proxmox cgroup/tap telemetry over SSH → results/*.jsonl
  generate_graph.py          Per-metric average charts across runs
  generate_combined.py       CPU+egress combined charts
  generate_single.py         Single-run chart
  .env.exemple               Template: Proxmox creds + per-flavor client .exe paths
  results/ graphs/ Old\ Runs/ Raw NDJSON + rendered figures
```

Each server leg ships **two microVM build scripts** at its root (`aarch64` + `x86_64`), and Unreal
adds a `_trace` variant. They are self-contained temporary-image builders — they stage a native
server build into a minimal Linux rootfs and package a bootable initramfs (and, for x86_64, an ISO).

---

## 0. Prerequisites (per the scripts you intend to run)

| You want to…                                          | Need |
|-------------------------------------------------------|------|
| Build the Rust QUIC client cdylib for Unity           | Rust 1.75+ stable (`rustup`). No OpenSSL/MsQuic — quinn+rustls are pure Rust |
| Build the Unity DGS server / client                   | Unity 6 with the **Dedicated Server** build module |
| Build the Unreal DGS server / client                  | Unreal Engine **5.7** with the **Linux** platform + cross-compile toolchain |
| Build the Bevy server outside its microVM script      | `cargo-zigbuild` (`pip install ziglang` + `cargo install cargo-zigbuild`), or run in WSL |
| Build any **aarch64** microVM                         | Docker (`buildx` + `binfmt_misc` for cross-arch) **or** an aarch64 Linux host; `qemu-system-aarch64`, `cpio`, `gzip`, `curl` |
| Build any **x86_64** microVM                          | Docker (`buildx`, native); `qemu-system-x86_64`, `cpio`, `gzip`, `curl` |
| Produce an uploadable **ISO** from an x86_64 microVM  | `grub-pc-bin`, `grub-efi-amd64-bin`, `xorriso`, `mtools` |
| Run the benchmark harness                             | Python 3.10+, `pip install paramiko python-dotenv matplotlib numpy`; a Proxmox host reachable over SSH |

---

## 1. Build the native QUIC plugin (Unity only)

The Unity client and Unity DGS server both talk QUIC through `native/quic_client/` — a Rust cdylib
(`quinn` + `rustls`) exposing a C ABI that `Assets/_Game/Net/Quic/QuicNetworkClient.cs` and
`QuicNetworkServer.cs` load via `[DllImport("dgsvshs_socket")]`. It implements the routing in
`Assets/_Game/Net/WireFormat.md` (reliable tags → bidi stream, unreliable tags → QUIC datagram).
**Build it once per host platform before pressing Play in the Editor.**

**Windows**
```powershell
cd native/quic_client
.\scripts\build_and_deploy.ps1     # cargo build --release --lib, then copies the .dll into Unity
```

**Linux / macOS**
```bash
cd native/quic_client
./scripts/build_and_deploy.sh      # produces libdgsvshs_socket.so / .dylib
```

Both scripts drop the library into `DGSvsHS/Assets/Plugins/x86_64/`. Unity resolves natives by base
name (`dgsvshs_socket`), so the same DllImport works on all three desktop platforms. To
cross-compile (e.g. a Linux `.so` from Windows): `cargo install cross && cross build --release --lib
--target x86_64-unknown-linux-gnu`.

---

## 2. Unity leg — client + DGS server (build template)

Open `DGSvsHS/` in Unity 6 (Dedicated Server module installed). The "build template" is three
coupled project settings, all driven by one menu:

**Menu: `DGSvsHS → Build Mode → …`** (`Assets/_Game/Editor/BuildModeSwitcher.cs`)

| Mode | Defines | Client default port | Purpose |
|------|---------|:-------------------:|---------|
| **DGS — Unity DOTS Server** | `WITH_DGS` | 7777 | The Unity DOTS leg |
| **HS/Bevy — Rust server** | `WITH_HS`, `HS_TARGET_BEVY` | 4433 | Unity client pointed at the Bevy server |
| **HS/Arch** *(legacy)* | `WITH_HS`, `HS_TARGET_ARCH` | 7778 | Deprecated — not in this study |
| **BareBone — Minimal Listener** | `WITH_BAREBONE` | 7779 | Green-computing idle baseline |

Switching a mode does three things at once, for **both** the Standalone and Server build targets:
1. rewrites the scripting-define symbols (mutually exclusive),
2. sets managed stripping to **Low** (matched across modes so stripping can't confound the compare),
3. patches the serialized `ClientMain.Port` inside `Assets/Scenes/Client.unity` so the Inspector
   already points at the right server.

**Building each artifact** (Unity Build Profiles):

- **DGS server** → pick `Build Mode → DGS`, then a **Dedicated Server** profile
  (`Server-Linux-x86_64` / `Server-Linux-ARM64` / `Server-Windows`), scene list = **only**
  `Assets/Scenes/Server.unity`. Output must contain the entrypoint binary, `UnityPlayer.so`/`.dll`,
  and `*_Data/` — the microVM scripts in §5 expect exactly this layout. Local sanity run:
  `Build/<date>/Server/Windows/DGSvsHS.exe` → watch for
  `[DedicatedServerMain] Listening on port 7777, seed C0FFEEF00D`.
- **Unity client** → pick `Build Mode → DGS` (to test against Unity DGS) or `HS/Bevy`, then a
  **Standalone** profile (`Client-Windows`), scene list = `Assets/Scenes/Client.unity`.

The client autopilot (`Assets/_Game/Harness/TrialRunner.cs`) accepts:
```
--server <ip>   --port <p>   --bot-id 0..3   --seed <N>
--duration <sec>   --warmup <sec>   --output trial.ndjson
```

---

## 3. Unreal leg — client + MASS server

Open `Unreal/UnrealvsHS/UnrealvsHS.uproject` in UE 5.7. The MASS simulation lives in
`Source/UnrealvsHS/Server/` (`SimRunner`/`SimSystems`/`SimContext`), MASS fragment types in `Mass/`,
and the MsQuic transport in `Net/` (using the bundled `Source/ThirdParty/MsQuic/`). The Unreal client
+ deterministic autopilot are in `Source/UnrealvsHS/Client/` — `UvHSClientGameMode.cpp` parses both
UE-native `-key=value` flags **and** Unity-style `--bot-id N` long flags so the same harness drives it.

**Server build** (for the microVM in §5): UE Editor → **Platforms → Linux → Cook Content + Package
Server**. The staged output must contain `<Project>Server.sh` at its root and
`<Project>/Binaries/Linux/<Project>Server` (Shipping/Test/Debug suffixes are auto-detected). The
microVM launches it with `-QuicPort=7780 -nullrhi -nosound -unattended`.

**Client build**: package a **Windows** client for driving trials (`UnrealvsHS.exe`).

---

## 4. Bevy leg — Rust server

Cargo workspace under `rust/` (`cli` + `gameplay` + `stress_client`). The server binary is `cli`,
which calls `gameplay::launch_server`. Physics is avian2d on Bevy's stock ECS scheduler (intentionally
un-tuned, per the paper's threats-to-validity).

```bash
cd rust
cargo run -p cli --release                      # run locally (listens on QUIC 4433)
cargo run -p cli --release -- --seed 0xC0FFEEF00D --duration 300
cargo run -p cli --release --features tracy     # stream Bevy spans to a Tracy profiler
```

Server CLI: `--seed=0xHEX|N` (default `0xC0FFEEF00D`), `--duration=SEC`, `--god-mode` (disable enemy
contact damage). On Windows, `boring-sys`/musl builds need WSL or `cargo-zigbuild`; the microVM
script handles this for you.

---

## 5. MicroVM builds (all three legs)

Each leg's `build_microvm_<arch>.sh` stages a native server build into a minimal Linux rootfs and
emits, into a leg-local `.microvm_<arch>/`:

- `vmlinuz-virt` — Alpine `linux-virt` kernel (auto-discovered latest, ~10 MB)
- `initramfs.cpio.gz` — rootfs + custom `/init` (Bevy ~5 MB · Unreal/Unity hundreds of MB)
- **x86_64 only:** a hybrid BIOS+UEFI bootable `*-microvm.iso` via `grub-mkrescue`

All scripts bake a **static IP** into `/init`. Defaults `192.168.0.205/24`, gateway `192.168.0.1`,
DNS `8.8.8.8`. Override per build:
```bash
STATIC_IP=192.168.1.50 STATIC_GATEWAY=192.168.1.1 ./build_microvm_x86_64.sh
```

| Leg | Base image | Forwarded port | Notes |
|-----|-----------|:--------------:|-------|
| **Unity DGS** (`DGSvsHS/`) | Debian bookworm-slim (glibc — `UnityPlayer.so` won't run on musl) | 7777 | Auto-locates the latest `Build/.../UnityPlayer.so`; installs the system libs Unity dlopens |
| **Unreal DGS** (`Unreal/UnrealvsHS/`) | Debian bookworm-slim | 7780 | Auto-locates `.../Binaries/Linux/<Project>Server`; strips `*.debug/*.sym`; runs UE as non-root uid 1000 (UE5 refuses root) |
| **Bevy** (`rust/`) | Alpine (static musl) | 4433 | `cargo zigbuild --target <arch>-unknown-linux-musl`; the binary **is** `/init` — no Dockerfile needed. Flags: `--god-mode`, `--tracy` |

```bash
# examples — run from the leg's own directory
cd DGSvsHS            && ./build_microvm_x86_64.sh            # auto-locate latest Unity build
cd Unreal/UnrealvsHS  && ./build_microvm_x86_64.sh /abs/path/to/StagedBuild
cd rust               && ./build_microvm_x86_64.sh --god-mode
```

The aarch64 scripts boot the image immediately in QEMU (`-machine virt`, `accel=hvf`/`kvm`) with the
leg's port host-forwarded. The x86_64 scripts additionally emit the ISO: upload via **Proxmox UI →
Storage → ISO Images → Upload**, create a VM with CD/DVD pointing at it, and start. Serial-console
logs land in the Console tab; the server's own log is at `/tmp/{unity,unreal}.log`.

---

## 6. Running the benchmark (`test_harness/`)

The harness deploys each server as a Linux microVM on one **Proxmox VMID** with identical guest specs,
then samples per-process cgroup metrics over SSH while the deterministic clients hammer it.

**Setup:** copy `test_harness/.env.exemple` → `test_harness/.env` and fill in:
```ini
PROXMOX_HOST="192.168.0.100"
PROXMOX_USER="root"
PROXMOX_PASSWORD="…"
CLIENT_EXE_DGS="…/Client-DGS/DGSvsHS.exe"
CLIENT_EXE_BEVY="…/Client-Bevy/DGSvsHS.exe"
CLIENT_EXE_UNREAL="…/Client/Unreal/Windows/UnrealvsHS.exe"
```

**Run a full case** (restarts the VM, records telemetry, launches autopilot clients, holds the active
window, then a cooldown; repeats `--runs` times → `results/<case>_N.jsonl`):
```bash
cd test_harness
python run_case.py --case dgs_baseline    --flavor dgs    --vmid 202 --runs 10 -r
python run_case.py --case bevy_stress      --flavor bevy   --vmid 200 --runs 10 --bots 4 -r
python run_case.py --case unreal_baseline  --flavor unreal --vmid 203 --runs 10 -r
```

Flags: `--flavor {dgs,bevy,unreal}` (`arch` is legacy), `--vmid` (Proxmox guest), `--runs`,
`--bots 1..4`, `--duration-sec` (default 300), `--cooldown-sec` (default 30), `-r/--restart`,
`--hz` (telemetry rate, default 20). The paper used **10 runs × 300 s** per case.

Per-flavor server ports the clients dial (`run_case.py::FLAVOR_PORT`): `dgs 7777 · bevy 4433 · unreal
7780` (`arch 7778`, legacy). Each 20 Hz sample is a JSON line: `{t, c=CPU% of one core, m=mem bytes,
rx, tx}` — CPU at fixed clock is the paper's power proxy.

**Analysis:**
```bash
python generate_graph.py       # per-metric average charts across all runs → graphs/averages/
python generate_combined.py    # CPU + network-egress combined → graphs/combined/
python generate_single.py dgs_baseline_1   # one run → graphs/individual/
```

---

## 7. Reproducing the paper end-to-end

1. Build the native QUIC plugin (§1).
2. Build all three servers as Linux microVMs (§2–§5) and upload/import each as a Proxmox VM with
   identical CPU/RAM/NIC specs, one VMID each.
3. Build the Unity **and** Unreal clients (§2, §3) on a **separate** host from the servers.
4. For each case in the connectivity matrix — Unity DGS ×Unity client, Unreal DGS ×Unreal client,
   Bevy ×Unity client, and Bevy ×Unreal client (cross-client invariance) — run
   `python run_case.py --case <name> --flavor <leg> --vmid <id> --runs 10 -r` (§6).
5. Render figures with `generate_graph.py` (§6). Expected ordering: Bevy lowest through the early/mid
   rounds, Bevy/Unity parity near round 9 (~25k entities), Unity ahead by round 10 (~35k), Unreal
   saturating a core early and dropping ticks at high mass.

See the PDF for full methodology, the scaling math (§IV), and threats to validity (§VI).
