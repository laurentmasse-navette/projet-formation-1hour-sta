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

- Install a WSL2 Linux distribution (to be chosen, for example Ubuntu LTS) if not already present.
- Move the repository from the Windows drive into `~/projects/projet-formation-1hour-sta` and reopen it in Cursor via WSL.

## Open decisions

- D0.2 Linux distribution and compiler versions (GCC and/or Clang).
- D0.3 C++ standard (C++17 or C++20).
- D0.4 Build system details (CMake presets, Ninja, warning levels, sanitizer builds, configurations).
- D0.5 Dependencies: Tcl 8.6 sourcing and version pinning, unit test framework, any other libraries.
- D0.6 Parser strategy (hand-written recursive descent or generator).
- D0.7 Repository layout, naming, namespaces, formatting and linting configuration.
- D0.8 Error and message policy (message IDs, exceptions or error codes at module boundaries).
