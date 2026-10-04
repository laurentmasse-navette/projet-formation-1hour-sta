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

- Done: install the WSL2 distribution chosen in D0.2 (`wsl --install -d AlmaLinux-8`); AlmaLinux 8.10 is in use.
- Done: move the repository from the Windows drive into `~/projects/projet-formation-1hour-sta` (by `git clone`)
  and reopen it in Cursor via WSL.

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

## D0.3 C++ standard and compilers: C++20, gcc-toolset-14 (Accepted, 2026-10-04)

### Decision

- Language standard: **C++20** (`CMAKE_CXX_STANDARD 20`, extensions off). C++20 modules are not used.
- Production compiler: **`gcc-toolset-14`** (GCC 14.2.1, already installed).
- **Clang 21** (AppStream) is a secondary compiler used in CI and for `clang-tidy`; it is not a release compiler.

### Rationale

- GCC 14 implements the C++20 core language and library features the project needs: `std::span`, concepts,
  ranges, `std::format`, designated initializers.
- A second compiler catches non-portable code early, in line with D0.1.
- `gcc-toolset-15` is available but offers nothing required; upgrading later is a low-risk change.

### Follow-up actions

- Done (2026-10-05): a `std::format` / `std::span` probe built with `gcc-toolset-14` links only the system
  `libstdc++` (GCC 8.5) and runs with the toolset absent from the environment. It requires at most
  `GLIBCXX_3.4.21`, `CXXABI_1.3.9`, `GLIBC_2.26`; stock AlmaLinux 8 provides `GLIBCXX_3.4.25` and glibc 2.28.
  Not yet repeated on a machine without the toolset installed (no container runtime available).

## D0.4 Build system: CMake presets with Ninja (Accepted, 2026-10-04)

### Decision

- **CMake >= 3.26** (3.26.5 installed) with a committed `CMakePresets.json`.
- Generator: **Ninja**, installed from the PowerTools repository (`ninja-build`).
- Presets: `debug`, `release`, `relwithdebinfo`, `asan-ubsan`.
- Warnings: `-Wall -Wextra -Wpedantic -Wshadow -Wconversion`; `-Werror` enabled in CI only.

### Follow-up actions

- `sudo dnf config-manager --set-enabled powertools && sudo dnf install ninja-build`.
- Install sanitizer runtimes: `gcc-toolset-14-libasan-devel`, `gcc-toolset-14-libubsan-devel`.

## D0.5 Dependencies: system Tcl 8.6, GoogleTest (Accepted, 2026-10-04)

### Decision

- **Tcl 8.6** from the system `tcl-devel` package (8.6.8 installed), located with `find_package(TCL)`.
  Tcl 9 is not supported (its C API differs); migration is a later decision.
- Unit tests: **GoogleTest**, fetched with CMake `FetchContent` at a pinned release tag (EPEL is not enabled).
- No other third-party libraries for milestone 1. Any addition requires a new decision entry.

## D0.6 Parser strategy: hand-written (Accepted, 2026-10-04)

### Decision

- Each format (Liberty, Verilog, SDF, later SPEF) has a **hand-written lexer and recursive-descent parser**.
- SDC is not parsed: it is executed as Tcl commands by the shell.

### Rationale

- Liberty, SDF, and SPEF have simple, regular grammars; the structural Verilog subset is small.
- No generator build dependency (flex/bison), precise error messages with file and line, and easier
  incremental subset growth.

## D0.7 Repository layout and conventions (Accepted, 2026-10-04)

### Decision

- Scope: top-level layout and coding conventions only. The module tree inside `src/` is decided in roadmap step 2.
- Top-level directories: `src/`, `tests/`, `testdata/`, `docs/`, `cmake/`.
- Root C++ namespace `sta`, with one nested namespace per module (finalized in step 2).
- Formatting and linting: committed `.clang-format` and `.clang-tidy` (Clang 21 tools).
- The detailed naming and style rules are recorded as a Cursor rule (coding conventions).

### Follow-up actions

- Done (2026-10-05): `.cursor/rules/coding-conventions.mdc`, applied to C++ and CMake files. Choices:
  Google-style naming (PascalCase types and functions, snake_case variables, trailing `_` members, `kName`
  constants), PascalCase file names with `.hpp` / `.cpp`, `#pragma once`, 2-space indentation, 80 columns.
- Write `.clang-format` and `.clang-tidy` matching the rule (with the first code-generating step).

## D0.8 Error and message policy (Accepted, 2026-10-04)

### Decision

- Inside readers and the core, errors are reported with **exceptions** (a project exception hierarchy).
- At the Tcl command boundary, exceptions are caught and converted to **Tcl errors** (`TCL_ERROR` with a message);
  no exception crosses the Tcl C API.
- Every user-visible message has a **project-specific ID** (`<AREA>-<NNN>`, for example `LIB-012`, `SDF-003`)
  and a severity (info, warning, error).
- A central message handler supports suppression and per-ID limits.

## D0.9 Version control: git on GitHub, trunk-based with pull requests (Accepted, 2026-10-05)

### Decision

- Remote: **GitHub**, `git@github.com:laurentmasse-navette/projet-formation-1hour-sta.git`, accessed over **SSH**.
  The former Windows-drive repository (`/mnt/j/...`) is no longer a remote.
- Branching: **trunk-based**. `main` is always in a working state; all work happens on short-lived branches
  merged through GitHub pull requests.
- Branch names: `step-NN/<topic>` for roadmap step work (for example `step-04/liberty-reader`),
  `docs/<topic>` and `fix/<topic>` for work outside a step. Lowercase, words separated by hyphens.
- Commit messages: imperative subject line of at most 72 characters, optionally prefixed with the decision or
  step (`D0.9: ...`, `Step 4: ...`); a blank line, then a body wrapped at 72 characters explaining why.
- Merge strategy: **squash merge**, one commit on `main` per pull request; the pull request title follows the
  commit subject rules.
- Protection of `main`: pull requests required now; required CI status checks added once D0.10 is settled.
- Line endings: `.gitattributes` with `* text=auto eol=lf`, so text files use LF on every platform.

### Rationale

- Pull requests give a review point for each change, including agent-generated ones, and a natural hook for CI.
- Step-prefixed branch names tie history to the roadmap; squash merges keep `main` linear, one entry per change.
- Enforcing LF avoids whole-file diffs when the repository is touched from Windows tools (CRLF conversion).

### Follow-up actions

- Enable branch protection (or a ruleset) on `main` in the GitHub repository settings: require a pull request
  before merging; allow squash merging only.
- Add required status checks once CI exists (D0.10).

## Open decisions

- D0.10 Continuous integration: platform, build matrix (GCC release build, Clang build, sanitizers), triggers.
- D0.11 Project license, consistent with the intellectual property rule.
