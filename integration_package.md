# C++ System Application — Integration Package

This document bundles:

* Repository layout and file templates
* `CMakePresets.json` template
* GitHub Actions CI pipeline template
* Integration diagrams (ASCII) and deployment flow
* Developer onboarding guide
* Branching strategy & release process

---

# 1. Repository layout (recommended)

```
/ (root)
├─ .github/
│  └─ workflows/ci.yml               # CI pipeline (GitHub Actions)
├─ cmake/                            # Shared CMake modules and helper macros
│  └─ Find*.cmake
├─ docs/                             # Architecture, ADRs, onboarding, diagrams
│  ├─ architecture.md
│  ├─ onboarding.md
│  └─ RELEASE_NOTES.md
├─ include/                          # Public headers (install target headers)
│  └─ project_name/
├─ src/                              # Private implementation files
│  ├─ core/
│  └─ main/
├─ tests/                            # Unit and integration tests
│  ├─ unit/
│  └─ integration/
├─ third_party/                      # Vendored dependencies (if any)
├─ tools/                            # Developer tooling and scripts
├─ scripts/                          # Build, lint, format helper scripts
├─ CMakeLists.txt                    # Top-level cmake
├─ CMakePresets.json                 # CMake configure/build presets
├─ .clang-format
├─ clang-tidy.yaml
├─ .gitignore
└─ README.md
```

Notes:

* Keep public headers under `include/` matching install layout so `install()` works smoothly.
* Use `cmake/` for reusable macros like `project_package_config()` and `enable_coverage()`.

---

# 2. `CMakePresets.json` (template)

```json
{
  "version": 3,
  "configurePresets": [
    {
      "name": "configure-debug",
      "displayName": "Configure Debug",
      "description": "Debug config with tests and sanitizers",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/debug",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug",
        "ENABLE_TESTING": "ON",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON",
        "ENABLE_SANITIZERS": "ON"
      }
    },
    {
      "name": "configure-release",
      "displayName": "Configure Release",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/release",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Release",
        "ENABLE_TESTING": "OFF",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "OFF"
      }
    }
  ],
  "buildPresets": [
    {
      "name": "build-debug",
      "configurePreset": "configure-debug",
      "jobs": 8
    },
    {
      "name": "build-release",
      "configurePreset": "configure-release",
      "jobs": 8
    }
  ],
  "testPresets": [
    {
      "name": "test-debug",
      "configurePreset": "configure-debug"
    }
  ]
}
```

Usage examples:

* `cmake --preset configure-debug`
* `cmake --build --preset build-debug`
* `ctest --preset test-debug --output-on-failure`

---

# 3. GitHub Actions CI template (`.github/workflows/ci.yml`)

```yaml
name: CI

on:
  pull_request:
    branches: [ dev, main ]
  push:
    branches: [ dev, main ]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        config: [ debug, release ]
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Set up CMake
        uses: jwlawson/actions-setup-cmake@v3

      - name: Install dependencies (vcpkg)
        run: |
          sudo apt-get update && sudo apt-get install -y build-essential ninja-build
          git clone --depth 1 https://github.com/microsoft/vcpkg.git vcpkg
          ./vcpkg/bootstrap-vcpkg.sh
          ./vcpkg/vcpkg install gtest

      - name: Configure
        run: |
          if [ "${{ matrix.config }}" = "debug" ]; then cmake --preset configure-debug; else cmake --preset configure-release; fi

      - name: Build
        run: cmake --build --preset build-${{ matrix.config }}

      - name: Run tests (debug only)
        if: matrix.config == 'debug'
        run: ctest --preset test-debug --output-on-failure

      - name: Run clang-tidy
        if: matrix.config == 'debug'
        run: |
          cmake --build build/debug --target clang-tidy || true

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ matrix.config }}
          path: |
            build/${{ matrix.config }}
```

Notes:

* Adjust dependency install steps for your chosen package manager (vcpkg, conan).
* Add caching for vcpkg or conan to speed up CI.

---

# 4. Integration diagrams (ASCII & flows)

## 4.1 High-level component diagram

```
  +----------------------+      +--------------------+
  |    CLI / UI         | <--> |    Core Logic      |
  +----------------------+      +--------------------+
             |                             |
             v                             v
        +----------+                 +-----------+
        | Storage  | <---SQL/Files---|  Infra    |
        +----------+                 +-----------+
```

## 4.2 CI/CD flow

```
Developer push -> PR -> GitHub Actions CI
    -> build & tests pass -> merge to dev
    -> nightlies / integration jobs run -> promote to main
    -> release job tags & publishes artifacts
```

## 4.3 Deployment flow (server)

```
CI artifacts (tar.gz / deb) --> staging server
  -> smoke tests -> manual approval -> production rollout
  -> systemd unit deployed + metrics verified
```

---

# 5. Onboarding guide (`docs/onboarding.md`)

## Quick start (developer machine)

1. Install prerequisites

   * CMake >= 3.26
   * Ninja
   * A supported compiler (GCC, Clang, MSVC)
   * vcpkg or conan
2. Clone the repo

   ```bash
   git clone git@github.com:example/your-project.git
   cd your-project
   ```
3. Install dependencies (vcpkg example)

   ```bash
   ./vcpkg/bootstrap-vcpkg.sh
   ./vcpkg/vcpkg install gtest fmt
   ```
4. Configure & build (debug)

   ```bash
   cmake --preset configure-debug
   cmake --build --preset build-debug
   ```
5. Run tests

   ```bash
   ctest --preset test-debug --output-on-failure
   ```

### Local development tips

* Use `clang-format` integration with your editor to keep style consistent.
* Use `compile_commands.json` (enabled in presets) to power IDE features.
* Run `clang-tidy` locally via the CMake `clang-tidy` target for quick checks.

### How to run integration tests

* Integration tests are orchestrated by the top-level CMake test target.
* Use environment files `tests/.env` (not committed) to configure credentials/paths.

### Contributing

* Create a feature branch off `dev`: `feature/your-feature-name`
* Open a PR targeting `dev` with description and test plan
* Ensure CI passes, address review comments
* When ready, a maintainer will merge to `dev` and trigger integration tests

---

# 6. Branching strategy & release process

## Branches

* `main` — production-ready; protected; only release merges
* `dev` — integration branch; PRs merge here by feature
* `feature/*` — short-lived feature branches
* `hotfix/*` — urgent fixes off `main`

## Pull request requirements

* Target branch: `dev` (unless hotfix)
* At least one approving review
* All CI checks pass
* Changelog entry (in `docs/RELEASE_NOTES.md` or PR body)

## Releases

1. When `dev` is stable, create a `release/x.y.z` branch
2. Run full integration & smoke tests
3. Create tag `vX.Y.Z` on `main` and merge `release/*` into `main`
4. CI publishes artifacts for the tag
5. Update `docs/RELEASE_NOTES.md` and the repo `CHANGELOG.md`

Versioning follows SemVer.

---

# 7. Suggested scripts (short list)

* `scripts/format.sh` — run clang-format over the repo
* `scripts/check_format.sh` — CI-style format check
* `scripts/run_clang_tidy.sh` — run clang-tidy
* `scripts/build_all_presets.sh` — build debug & release
* `scripts/create_release.sh` — helper that tags, builds, uploads

---

# 8. Next steps & customization points

* Add caching to CI (vcpkg/conan cache)
* Add security scans (CodeQL)
* Wire metrics and logging libraries
* Add systemd unit templates in `deploy/`

---

# 9. Appendix: Example `README.md` skeleton

```
# Project Name

Short description.

## Build
See `CMakePresets.json` for common presets:

```

cmake --preset configure-debug
cmake --build --preset build-debug

```

## Contributing
See `docs/onboarding.md` and follow the branch policy.
```

---

*End of document.*
