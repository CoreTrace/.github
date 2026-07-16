<div align="center">

# CoreTrace

**Compiler-grade C/C++ analysis, from source code to actionable evidence**

Static analysis · Runtime instrumentation · Cross-TU reasoning · JSON/SARIF reporting · CI policy gates

[![Licenses](https://img.shields.io/badge/Licenses-repository--specific-informational)](#licensing)
[![C++20](https://img.shields.io/badge/C%2B%2B-20-00599C?logo=cplusplus)](https://isocpp.org/)
[![LLVM](https://img.shields.io/badge/LLVM%2FClang-20-262D3A?logo=llvm)](https://releases.llvm.org/20.1.0/)

</div>

## What is CoreTrace?

CoreTrace is a modular analysis ecosystem for teams building security-sensitive and resource-constrained C and C++ software. It combines compiler-level static analysis, runtime instrumentation, reproducible reports, and CI integration to expose stack, memory, control-flow, and concurrency risks before release.

The project is intentionally split into independent components. The CLI orchestrates the workflow, the compiler layer owns Clang/LLVM integration, and each analyzer remains focused on one class of evidence. This separation keeps the system extensible and allows every tool to be used on its own or as part of the complete pipeline.

## CoreTrace Analyzer — the primary workflow

**CoreTrace Analyzer** is the main product experience. It is assembled from the [`coretrace` orchestrator](https://github.com/CoreTrace/coretrace), [`coretrace-compiler`](https://github.com/CoreTrace/coretrace-compiler), and the specialized analyzers rather than being implemented as one monolithic executable.

It:

- consumes C/C++ sources, compilation databases, and externalized analysis models;
- produces LLVM IR or instrumented binaries through the compiler layer;
- runs stack/resource, concurrency, and runtime analyses;
- reasons across translation units when project-wide context is required;
- normalizes findings into human-readable, JSON, and SARIF artifacts for local review, IDEs, and CI.

## Architecture

```text
 C/C++ sources · compile_commands.json · analysis models
                         │
                         ▼
              ┌─────────────────────┐
              │    coretrace CLI    │
              │ config · orchestration
              │ reporting · policy │
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │ coretrace-compiler  │
              │ Clang frontend      │
              │ LLVM IR · instrumentation
              └───────┬────────┬────┘
                      │        │
           LLVM IR    │        │ instrumented program/events
          ┌───────────┴───┐    └──────────────────────┐
          ▼               ▼                           ▼
 ┌────────────────┐ ┌────────────────┐     ┌────────────────┐
 │ stack/resource │ │  concurrency   │     │    runtime     │
 │    analyzer    │ │    analyzer    │     │    analyzer    │
 └────────┬───────┘ └────────┬───────┘     └────────┬───────┘
          └──────────────────┴───────────────────────┘
                              │
                              ▼
               Human output · JSON · SARIF · traces
                              │
                              ▼
                     CI · IDE · GUI · audits

 Cross-cutting: coretrace-log · coretrace-testkit · CI adapters · tool template
```

This layered design avoids coupling analyzer logic to delivery concerns. Compiler-version integration is isolated in `coretrace-compiler`, analyzers exchange explicit artifacts instead of reaching into one another, and reporting remains at the orchestration boundary. New analyzers can therefore join the pipeline without duplicating frontend or CI logic.

### Engineering principles

- **Separation of concerns** — compilation, analysis, orchestration, and policy have distinct owners.
- **Composable pipeline** — tools work independently and together through explicit artifacts.
- **CI-first evidence** — JSON and SARIF are first-class outputs, not afterthoughts.
- **Generic by default** — configuration and external models replace project-specific hardcoding.
- **Reproducibility** — versions, build context, normalized outputs, and benchmark conditions are recorded.

## Why LLVM/Clang 20?

CoreTrace uses LLVM because the project needs semantic program information, not only source-pattern matching.

- **Precise representation:** [LLVM IR](https://releases.llvm.org/20.1.0/docs/LangRef.html) is typed, low-level, and SSA-based. It exposes instructions, values, control flow, calls, and memory operations required for stack, resource, data-flow, and cross-function reasoning.
- **Faithful build context:** [Clang LibTooling](https://releases.llvm.org/20.1.0/tools/clang/docs/LibTooling.html) and compilation databases let tools reuse the real include paths, macros, language mode, and compiler options of the analyzed target.
- **Reusable artifacts:** IR can be handled in memory, serialized as bitcode, or inspected as text. This supports caching, debugging, deterministic comparisons, and integration between tools.
- **Modular analyses:** LLVM's [pass and analysis infrastructure](https://releases.llvm.org/20.1.0/docs/WritingAnLLVMNewPMPass.html) fits CoreTrace's component model: an analyzer can compute or consume explicit results without owning the complete compiler pipeline.
- **Appropriate scope:** the approach is well suited to compiled C and C++ systems where buffer access, stack use, recursion, uninitialized values, and interprocedural behavior must be inspected below the syntax level.

The trade-off is higher implementation complexity, LLVM-version coupling, and occasional loss of source-level intent. CoreTrace contains that cost in the compiler/adaptation layer and preserves source locations through compiler metadata wherever possible.

## Performance evidence

The following controlled measurements cover the cross-translation-unit pipeline in [`coretrace-stack-analyzer`](https://github.com/CoreTrace/coretrace-stack-analyzer). Lower elapsed time is better.

### Main results

| Isolated optimization | Before | After | Median improvement | 95% bootstrap CI of median time saved |
|---|---:|---:|---:|---:|
| Parallel execution, `jobs=1` → `jobs=4` | 373.2 ms | 253.9 ms | **−32.20% / 1.475×** | 115.9–123.5 ms |
| Global iteration → SCC worklist | 664.0 ms | 462.6 ms | **−30.38% / 1.437×** | 199.3–203.7 ms |

Each case was measured **25 times after 3 warmups**, using a deterministic interleaved order. Parallel-corpus throughput increased from **171.5 to 252.0 files/s**; cross-TU corpus throughput increased from **1,204.7 to 1,729.3 files/s**.

The broader comparison between `v0.6.0` and the current implementation shows a **43.55%** lower median time, from **452.1 to 253.9 ms**. This historical comparison is deliberately labelled **non-causal** because many changes separate the two versions.

### Causal controls

- **Parallelization:** the same commit and executable were used; only `jobs=1` versus `jobs=4` changed.
- **SCC worklist:** adjacent commits [`128e20c9`](https://github.com/CoreTrace/coretrace-stack-analyzer/commit/128e20c9bdaab11a7b19dc5f1e8c4abde17d4e8a) and [`6a8c64f6`](https://github.com/CoreTrace/coretrace-stack-analyzer/commit/6a8c64f6f64a141aa826c0585595be3113964e39) used exactly the same dependency versions, isolating the algorithm change.
- **Functional equivalence:** normalized JSON outputs were identical within every causal pair.

The SCC mechanism is also visible in internal timings:

| Internal signal | Before | After | Change |
|---|---:|---:|---:|
| Module constructions | 2,160 | 800 | **−62.96%** |
| Cross-TU resource phase | 296 ms | 107 ms | **−63.85%** |
| Total elapsed time | 664.0 ms | 462.6 ms | **−30.38%** |

### Memory, tests, and robustness

- Parallel execution costs approximately **+432 KiB** between `jobs=1` and `jobs=4`.
- The historical `v0.6.0` → current comparison shows approximately **+130 MiB RSS**. It is not attributed to multithreading alone: the current version contains more analyses, caches, and cross-TU structures. This is documented as a trade-off.
- Existing CTest suite: **1/1 passed**.
- Resilience scenarios: **3/3 passed** on `v0.6.0` and **5/5 passed** on the current version, including a missing file, invalid IR, unsupported format, `jobs=0`, and `SIGINT` interruption.
- All **125 measured processes** completed successfully.
- No existing test file was modified.
- Environment: **LLVM/Clang 20.1.2**, Release build, with machine and tool versions recorded.

## Repository map

### Core platform and analyzers

| Repository | License | Responsibility |
|---|---|---|
| [`coretrace`](https://github.com/CoreTrace/coretrace) | Apache-2.0 | Main CLI, orchestration, unified reports, tool invocation, and server mode |
| [`coretrace-compiler`](https://github.com/CoreTrace/coretrace-compiler) | Apache-2.0 | Clang-based compiler wrapper, LLVM IR production, builds, and instrumentation |
| [`coretrace-stack-analyzer`](https://github.com/CoreTrace/coretrace-stack-analyzer) | Apache-2.0 | Stack, recursion, resource, and cross-TU static analysis |
| [`coretrace-concurrency-analyzer`](https://github.com/CoreTrace/coretrace-concurrency-analyzer) | Apache-2.0 | Shared-state, lock, thread-lifecycle, and atomic-usage analysis |
| [`coretrace-runtime-analyzer`](https://github.com/CoreTrace/coretrace-runtime-analyzer) | Not declared | Runtime allocation, bounds, shadow-memory, call, and vtable diagnostics |
| [`coretrace-tool-template`](https://github.com/CoreTrace/coretrace-tool-template) | Not declared | Standard architecture and CI template for new CoreTrace tools |

### Developer experience and infrastructure

| Repository | License | Responsibility |
|---|---|---|
| [`coretrace-gui`](https://github.com/CoreTrace/coretrace-gui) | Apache-2.0 | Web and desktop interface |
| [`coretrace-vscode`](https://github.com/CoreTrace/coretrace-vscode) | GPL-3.0 | VS Code integration |
| [`coretrace-log`](https://github.com/CoreTrace/coretrace-log) | MIT | Lightweight, thread-safe C++20 logging library |
| [`coretrace-testkit`](https://github.com/CoreTrace/coretrace-testkit) | GPL-3.0 | Isolated tool execution and output-validation framework |
| [`coretrace-ci-consumer-demo`](https://github.com/CoreTrace/coretrace-ci-consumer-demo) | Not declared | External-consumer CI reference for packaging and compatibility checks |

### Research, experiments, and legacy

| Repository | Status | License | Scope |
|---|---|---|---|
| [`coretrace-apex`](https://github.com/CoreTrace/coretrace-apex) | Active | GPL-3.0 | Cross-platform PE and ELF binary analysis |
| [`coretrace-TscanCode`](https://github.com/CoreTrace/coretrace-TscanCode) | Active | Not identified | Static analysis experiments for C++, C#, and Lua |
| [`coretrace-web`](https://github.com/CoreTrace/coretrace-web) | Active | Not declared | Web experiments |
| [`coretrace-fuzz`](https://github.com/CoreTrace/coretrace-fuzz) | Active | Not declared | Fuzzing experiments |
| [`coretrace-qt`](https://github.com/CoreTrace/coretrace-qt) | Archived | Not declared | Historical Qt client |
| [`coretrace-vscode`](https://github.com/CoreTrace/coretrace-vscode) | Active | GPL-3.0 | UI for vscode |

## Tags and versioning

CoreTrace repositories use **independent versioning**: a tag identifies the state of one repository and does not imply that every component has the same release number. Consumers should pin the exact repository tag or commit SHA used by their build.

Version tags found through the GitHub API on **2026-07-16**:

| Repository | Latest version tag | Tagged commit date (UTC) | Version tags | Line |
|---|---:|---:|---:|---|
| [`coretrace` tags](https://github.com/CoreTrace/coretrace/tags) | `v0.74.0` | 2026-03-27 | 10 | 0.x |
| [`coretrace-compiler` tags](https://github.com/CoreTrace/coretrace-compiler/tags) | `v0.7.0` | 2026-02-05 | 11 | 0.x |
| [`coretrace-stack-analyzer` tags](https://github.com/CoreTrace/coretrace-stack-analyzer/tags) | `v0.18.2` | 2026-06-04 | 32 | 0.x |
| [`coretrace-gui` tags](https://github.com/CoreTrace/coretrace-gui/tags) | `v5.0.2` | 2026-07-15 | 24 | 5.x |
| [`coretrace-vscode` tags](https://github.com/CoreTrace/coretrace-vscode/tags) | `v0.1.0` | 2026-04-09 | 1 | 0.x |
| [`coretrace-testkit` tags](https://github.com/CoreTrace/coretrace-testkit/tags) | `v0.1.0` | 2026-01-29 | 1 | 0.x |
| [`coretrace-qt` tags](https://github.com/CoreTrace/coretrace-qt/tags) | `v0.1.0-alpha` | 2025-05-11 | 1 | Archived alpha |

Repositories absent from this table had no SemVer-shaped tag at the snapshot date. Components in a `0.x` line may still introduce incompatible changes; check the repository release notes and compatibility constraints before upgrading.

## Quick start

```bash
# Run the orchestrated static and dynamic workflow
./ctrace --input main.cpp --entry-points=main --verbose --static --dyn

# Emit LLVM IR through the compiler wrapper
./cc -S -emit-llvm test.cc

# Run stack analysis in CI and export JSON + SARIF
python3 scripts/ci/run_code_analysis.py \
  --analyzer ./build/stack_usage_analyzer \
  --compdb ./build/compile_commands.json \
  --fail-on error \
  --json-out artifacts/stack-usage.json \
  --sarif-out artifacts/stack-usage.sarif
```

See each repository for build prerequisites and complete command-line options.

## Technology

| Area | Choice |
|---|---|
| Core implementation | C++20 |
| Compiler infrastructure | LLVM/Clang 20 |
| Build system | CMake |
| Automation outputs | JSON and SARIF |
| Delivery | CI policy gates, IDE integration, GUI, and standalone CLI tools |

## Documentation

- [CoreTrace wiki](https://github.com/CoreTrace/coretrace/wiki) — CLI usage and project documentation
- [Compiler wiki](https://github.com/CoreTrace/coretrace-compiler/wiki) — compiler and instrumentation workflows
- [Stack analyzer wiki](https://github.com/CoreTrace/coretrace-stack-analyzer/wiki) — analyzer configuration and integration

## Licensing

Licensing is repository-specific. The core platform and active C++ analyzers are primarily Apache-2.0; `coretrace-log` uses MIT; `coretrace-vscode`, `coretrace-testkit`, and `coretrace-apex` use GPL-3.0. A repository marked **Not declared** or **Not identified** must not be assumed reusable without further clarification. Always inspect the target repository's license before redistribution or integration.

## Contributing

Contributions are welcome through the issues and pull requests of the relevant repository. For cross-component changes, describe the affected artifact contracts and compatibility expectations explicitly.

<div align="center">

[Browse all repositories](https://github.com/orgs/CoreTrace/repositories) · [Read the documentation](https://github.com/CoreTrace/coretrace/wiki)

</div>
