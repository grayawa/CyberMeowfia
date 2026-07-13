# CyberMeowfia

This repository is a collection of vulnerability research and exploit-development subprojects. It is not a single buildable application, and the useful source of truth is usually the code and per-directory build logic rather than repo-level prose.

## IonStack followup, follow https://github.com/NebuSec/CyberMeowfia/issues/44

## Scope

The current public-facing work here is centered on:

- `IonStack/CVE-2026-43499/`
- `security-research/Linux-CVE-2026-23274/`
- `security-research/Linux-CVE-2026-43501/`
- `security-research/Chrome-CVE-2026-5865/`
- `IonStack/CVE-2026-10702/`

## Repository Layout

- `IonStack/CVE-2026-43499/exploit/`
  Android aarch64 preload-based exploit project. The tree contains shared logic under `src/` plus per-device / per-build overrides under `src/targets/`.
- `IonStack/CVE-2026-43499/poc/poc.c`
  Standalone proof-of-concept source.
- `security-research/Linux-CVE-2026-23274/`
  Separate Linux exploit project with its own build flow.
- `security-research/Linux-CVE-2026-43501/`
  Separate Linux exploit project with its own build flow.
- `IonStack/CVE-2026-10702/` and `security-research/Chrome-CVE-2026-5865/`
  Browser-side research snippets.

## Current Focus: `eureka-UP1A.231005.007.A1`

The main unpublished work that was cleaned up for release lives under:

- `IonStack/CVE-2026-43499/exploit/src/targets/eureka-UP1A.231005.007.A1/`

This target was used for research on an Oculus / Meta Quest 3 class Android 14 kernel based on Linux `5.10.240`. The objective was to adapt the existing project structure to a new target while preserving the original exploit split:

- target-specific constants and symbol-derived offsets in `target.h`
- target-local behavior overrides in `main.c`, `slide.c`, `fops.c`, `pipe.c`, `util.c`, `common.h`
- shared generic logic in `IonStack/CVE-2026-43499/exploit/src/`

## What Was Verified

During this work, the following points were checked against the local kernel image and an official `5.10.240` kernel source tree:

- The target kernel family is consistent with Linux `5.10.240`.
- The exploit build structure really does select per-target overrides first and then fall back to shared `src/` files.
- The preload library entrypoint is the constructor path in `src/preload.c`.
- The embedded `su_daemon` is a helper payload, not the primary library entrypoint.
- The `rt_mutex` / PI-related layout assumptions used by the target match the expected field order from the upstream-style `5.10.240` kernel sources closely enough that they were not the first suspected failure point.
- `struct rt_mutex_waiter` is a stack object in the relevant futex PI paths, which preserves the general shape assumed by the existing research approach.

## Research Conclusions So Far

The most important outcome of the `eureka-UP1A.231005.007.A1` bring-up was negative result triage: several directions that looked initially plausible were narrowed down or deprioritized.

### 1. The main blocker is still the slide / KASLR leak stage

The hardest part was not target-local constants alone. The real blocker was finding a leak primitive that survives this kernel / device environment closely enough to recover the kernel image base in a usable way.

### 2. A `pselect`-style leak path did not carry over cleanly

An external `slide.c` sample based on a `pselect`-driven stack copy was reviewed and compared with the target work. That code relies on a very specific overlap shape and syscall behavior. On this target, that assumption did not transfer cleanly enough to treat it as solved.

### 3. `process_vm_*` looked attractive but was the wrong object class

Inspection of the official `5.10.240` tree showed that the relevant `process_vm` path uses on-stack arrays of `struct page *`. That makes it a poor fit for the style of waiter-shaped overlap required by the existing leak design, even if timing or overlap can be achieved.

### 4. IPv6 setsockopt-based paths were not obviously invalid, but too weak overall

The IPv6 multicast path exposes stack-resident structures and was worth checking. In practice it still did not look like the right full-chain leak primitive for this target once stack depth, structure shape, and end-to-end exploit needs were considered together.

### 5. SELinux and access surface matter here

This environment is not just "generic Android". In testing, access to some interfaces used by other variants was blocked for the shell domain. That means even a promising memory-corruption stage can still run into policy and interface restrictions later in the chain.

### 6. Crash-log based fallback was not reliable in the tested setup

A panic / log-collection fallback was considered, but the expected post-crash logging surfaces were not available in a useful form in the observed device state.

## Why This Target Was Harder Than Expected

One of the recurring themes in this project is that "same bug family" does not imply "small retargeting job".

The difficult parts here were not just:

- symbol lookup
- offset porting
- a few structure-field updates

The real problems were target-specific environment constraints:

- the leak primitive that worked elsewhere was not obviously portable
- candidate syscalls produced the wrong stack objects or the wrong overlap shape
- the device policy surface was stricter than a naive desktop-kernel mental model would suggest

That is why this target took more elimination work than a straightforward "change offsets and rebuild" port.

## Build Notes

This repo intentionally keeps build logic with each subproject. For `IonStack/CVE-2026-43499/exploit/`, the Makefile confirms:

- the project is selected through `PROJECT=<target>`
- per-target source override is automatic when a file exists under `src/targets/<PROJECT>/`
- the Android NDK is discovered from environment variables or a local cache path
- `build/` is generated output and is not checked in

The public repo does not include device images, IDA databases, extracted symbols, OTA zips, or temporary cloned kernel trees that were used during local research.

## Release Checklist

Before publishing, confirm that the diff contains only:

- `README.MD`
- `AGENTS.md`
- `.gitignore`
- the intended files under `IonStack/CVE-2026-43499/exploit/src/targets/eureka-UP1A.231005.007.A1/`

and does not contain:

- `build/`
- `_tmp/`
- `.i64` databases
- OTA zips, `kernel.elf`, `symbol`, or similar local analysis artifacts

## Safety

This repository contains exploit research. It is provided for analysis, auditing, and historical research. It is not a step-by-step exploitation guide, and the removed local artifacts are intentionally not part of the published tree.

Do not run or adapt any of this against systems you do not own or explicitly control.

## Status

The `eureka-UP1A.231005.007.A1` target directory represents in-progress research rather than a claimed end-to-end successful exploit chain.

Current public state:

- the target-specific port has been split out cleanly under `src/targets/eureka-UP1A.231005.007.A1/`
- symbol-derived constants and the main target-local source overrides are present
- several candidate leak directions were investigated and narrowed down
- `rt_mutex` / waiter layout assumptions were cross-checked against a matching `5.10.240` kernel source tree
- local-only artifacts used during reverse engineering and testing have been removed from the published tree

What to do:

- Find another way to get KBase,directly bypassing slide stage?Or fix the slide.h,but with little hope.
- Write an app to execute this PoC,not using shell permission
- Upstream it
