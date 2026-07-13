# AGENTS

- This repo is a collection of exploit/research subprojects, not a single app. Trust per-directory `Makefile`s and source over repo-level prose when they conflict.
- There is no verified repo-wide test, lint, formatter, typecheck, or CI config. Do not claim a global validation step exists.

## High-Value Paths

- `IonStack/CVE-2026-43499/exploit/`: Android aarch64 preload exploit. Default `make` target is `preload`, which builds `build/<PROJECT>/bin/preload.so`.
- `IonStack/CVE-2026-43499/poc/poc.c`: standalone C PoC.
- `security-research/Linux-CVE-2026-23274/` and `security-research/Linux-CVE-2026-43501/`: separate Linux exploit builds with their own `Makefile`s.
- `IonStack/CVE-2026-10702/` and `security-research/Chrome-CVE-2026-5865/`: HTML/JS research snippets, no build system.

## Android Exploit Notes

- Work from `IonStack/CVE-2026-43499/exploit/`.
- Use `make list-projects` to see valid `PROJECT` values. The default is `blazer-CP2A.260605.012`.
- Use `make PROJECT=<name>` to build a specific target and `make info` to print the resolved toolchain and paths.
- The build chooses source files via `pick_src`: if `src/targets/<PROJECT>/<file>` exists, it overrides `src/<file>`. Check both places before editing shared logic.
- NDK discovery is automatic: `ANDROID_NDK_HOME`, then `ANDROID_NDK_ROOT`, then `~/android-ndk-cache/android-ndk-r29`. If the direct NDK compiler is unavailable, the Makefile falls back to host `clang` with Android target flags.
- The effective exploit entrypoint is `src/preload.c` via `__attribute__((constructor))`; `src/su_daemon.c` is built as an embedded helper binary, not the main library entrypoint.
- `build/` is generated output. Do not treat it as source.

## Linux Exploit Notes

- Work from the specific CVE directory; there is no shared top-level build.
- `make` builds `exploit`; `make exploit_debug` adds debug symbols; `make clean` removes outputs.
- First build bootstraps dependencies by downloading `target_db.kxdb`, cloning `google/kernel-research`, and building `kernel-research/libxdk`. Expect network access and generated local directories.
- `target_db.kxdb` and `kernel-research/` are disposable generated dependencies inside each CVE directory.

## Verification And Safety

- Preferred verification is building only the subproject you changed.
- Do not run exploit binaries or preload libraries unless the task explicitly requires execution; the checked-in sources warn they may modify system state or destabilize devices.
- Local reverse-engineering artifacts and build outputs are intentionally excluded from the public tree via `.gitignore`; do not treat ignored files as source.
