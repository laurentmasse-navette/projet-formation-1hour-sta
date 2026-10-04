# Step 0: Language and build decisions

Roadmap step: 0 (Planning). See [roadmap](../roadmap.md).

Status legend: **Accepted**, **Open**, **Superseded**.

## D0.1 Host platform: Linux via WSL2 (Accepted, 2026-10-04)

### Decision

- The primary development and target platform is **Linux**, run on Windows workstations through **WSL2**.
- The C++ code stays **portable** (standard C++ and portable libraries only, no Linux-only APIs outside an isolated
  platform layer), so that a native Windows build with **MSYS2** (MinGW-w64/UCRT) or **MSVC** remains possible later.
- The working copy lives **inside the Linux file system** at `~/projects/projet-formation-1hour-sta`,
  and is opened in Cursor through its WSL connection.

### Rationale

- EDA is a Linux-centric industry: vendor and open-source tools, scripts, and user expectations target Linux.
- WSL2 runs a real Linux kernel, so the tool is built and tested exactly as Linux users will build it, and Linux
  reference tools (for example OpenSTA or Yosys, used as black boxes) can run alongside it.
- Accessing Windows drives from WSL2 (`/mnt/<drive>/...`) goes through a slow file-sharing bridge; keeping sources
  in the Linux file system keeps builds and `git` fast.

### Alternatives considered

| Option | Why not chosen as primary |
|---|---|
| Cygwin | POSIX emulation layer, not Linux: cannot run Linux binaries, slower `fork` and file I/O, subtle behavior differences, binaries depend on `cygwin1.dll`. Rejected. |
| MSYS2 | Produces native Windows executables, but is not Linux. Kept as a candidate for a later Windows port. |
| Native MSVC | Diverges from the Linux-centric EDA ecosystem. Kept as a candidate for a later Windows port. |

### Follow-up actions

- Install the WSL2 distribution chosen in D0.2 (`wsl --install -d AlmaLinux-8`).
- Move the repository from the Windows drive into `~/projects/projet-formation-1hour-sta` and reopen it in Cursor via WSL.

## D0.2 Linux distribution: AlmaLinux 8 (Accepted, 2026-10-04)

### Decision

- The development and reference build platform is **AlmaLinux 8** (RHEL 8 compatible), run under WSL2.
- Production builds use a **`gcc-toolset-N`** compiler from AppStream (exact version decided with D0.3),
  not the system GCC 8.5.

### Rationale

- EDA customers run RHEL-compatible enterprise Linux, often lagging several releases behind; RHEL 8 is still
  a common deployment baseline. This matches established EDA development practice.
- Building on the **oldest supported platform** (glibc 2.28) yields binaries that also run on newer RHEL 9/10
  and other distributions, because glibc is backward compatible but not forward compatible.
- `gcc-toolset` compilers provide modern C++ support while producing binaries that depend only on the
  base system runtime libraries (newer `libstdc++` parts are linked statically), so they run on a stock RHEL 8.

### Consequences

- The system GCC 8.5 has incomplete C++17 library support and almost no C++20; the project must not rely on it.
  The C++ standard choice (D0.3) is bounded by the newest available `gcc-toolset`.
- Distribution packages are older (for example Tcl 8.6.8). Some tools come from AppStream, PowerTools/CRB, or EPEL
  (Ninja, GoogleTest), or are fetched by the build (D0.5).
- RHEL 8 maintenance support ends around mid-2029. Moving the baseline to AlmaLinux 9 is a planned future decision,
  not an emergency, as long as the code stays portable (D0.1).
- Optional later CI targets: AlmaLinux 9 and a recent Ubuntu LTS, to catch portability issues early.

### Alternatives considered

| Option | Why not chosen |
|---|---|
| Ubuntu 24.04 / 26.04 LTS | Newer toolchain and packages, but not a typical EDA customer platform; binaries built on a newer glibc may not run on RHEL 8. Kept as an optional CI target. |
| AlmaLinux 9 / Rocky 9 | Longer support window, but raises the minimum glibc to 2.34 and excludes RHEL 8 customers. Candidate for the next baseline. |

## Open decisions

- D0.3 C++ standard (C++17 or C++20) and `gcc-toolset` version; whether Clang is also supported.
- D0.4 Build system details (CMake presets, Ninja, warning levels, sanitizer builds, configurations).
- D0.5 Dependencies: Tcl 8.6 sourcing and version pinning, unit test framework, any other libraries.
- D0.6 Parser strategy (hand-written recursive descent or generator).
- D0.7 Repository layout, naming, namespaces, formatting and linting configuration.
- D0.8 Error and message policy (message IDs, exceptions or error codes at module boundaries).
