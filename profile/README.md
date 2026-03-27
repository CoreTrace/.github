<div align="center">

# CoreTrace

**LLVM-powered C/C++ analysis toolchain**

Static analysis · Runtime instrumentation · Developer diagnostics · CI policy gates

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://www.apache.org/licenses/LICENSE-2.0)
[![C++20](https://img.shields.io/badge/C%2B%2B-20-00599C?logo=cplusplus)](https://isocpp.org/)
[![LLVM](https://img.shields.io/badge/LLVM-20-262D3A?logo=llvm)](https://llvm.org/)

---

</div>

## What is CoreTrace?

CoreTrace is a modular toolchain that helps teams ship **safer and more predictable C/C++ systems**. It orchestrates multiple static and dynamic analysis tools into a unified pipeline, producing standardized reports (JSON, SARIF) ready for automation and security workflows.

## Repositories

### Core

| | Repository | Description |
|---|---|---|
| | Repository | License | Description |
|---|---|---|---|
| | [coretrace](https://github.com/CoreTrace/coretrace) | Apache 2.0 | Main CLI orchestrator — static/dynamic analysis, tool invocation, SARIF export, server mode |
| | [coretrace-compiler](https://github.com/CoreTrace/coretrace-compiler) | Apache 2.0 | Clang/LLVM compiler wrapper — IR emission, binary build, instrumentation modules |

### Analyzers

| | Repository | License | Description |
|---|---|---|---|
| | [coretrace-stack-analyzer](https://github.com/CoreTrace/coretrace-stack-analyzer) | Apache 2.0 | Stack and resource analysis engine with SARIF output and CI adapters |
| | [coretrace-concurrency-analyzer](https://github.com/CoreTrace/coretrace-concurrency-analyzer) | Apache 2.0 | Threading and race condition detection |

### Developer Tools

| | Repository | License | Description |
|---|---|---|---|
| | [coretrace-gui](https://github.com/CoreTrace/coretrace-gui) | Apache 2.0 | Web and desktop interface |
| | [coretrace-vscode](https://github.com/CoreTrace/coretrace-vscode) | Apache 2.0 | VS Code extension |

### Libraries & Infrastructure

| | Repository | License | Description |
|---|---|---|---|
| | [coretrace-log](https://github.com/CoreTrace/coretrace-log) | MIT | Lightweight C++20 logging library |
| | [coretrace-testkit](https://github.com/CoreTrace/coretrace-testkit) | Apache 2.0 | Python testing framework for the ecosystem |
| | [coretrace-ci-consumer-demo](https://github.com/CoreTrace/coretrace-ci-consumer-demo) | Apache 2.0 | CI/CD integration reference and demo |

### Documentation

| | Repository | Description |
|---|---|---|
| | [coretrace.wiki](https://github.com/CoreTrace/coretrace.wiki) | CLI usage and project documentation |
| | [coretrace-compiler.wiki](https://github.com/CoreTrace/coretrace-compiler.wiki) | Compiler wrapper documentation |
| | [coretrace-stack-analyzer.wiki](https://github.com/CoreTrace/coretrace-stack-analyzer.wiki) | Analyzer workflows and integration notes |

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                    coretrace CLI                      │
│          orchestration · config · SARIF export        │
├────────────┬─────────────────────┬───────────────────┤
│  compiler  │   stack-analyzer    │   concurrency-    │
│  wrapper   │                     │   analyzer        │
│  (Clang)   │   (LLVM analysis)   │   (thread safety) │
├────────────┴─────────────────────┴───────────────────┤
│              LLVM / Clang toolchain                   │
└──────────────────────────────────────────────────────┘
        ▲                                    │
        │ C/C++ source                       ▼
   Developer                          JSON / SARIF reports
                                      → CI gates, IDE, GUI
```

**Principles:**
- **Separation of concerns** — analysis engines stay independent from CI policy logic
- **Composable pipeline** — compiler, analyzers, and orchestration layer work independently or together
- **CI-first outputs** — JSON and SARIF are first-class artifacts
- **Generic by default** — externalized models and configuration over hardcoded behavior

## Quick Start

```bash
# 1. Run end-to-end analysis locally
coretrace analyze ./my-project

# 2. Use the compiler wrapper for IR-level workflows
coretrace-compiler build ./my-project

# 3. Add stack analysis in CI with SARIF output
coretrace-stack-analyzer --format sarif ./my-binary
```

## Tech Stack

| | |
|---|---|
| **Language** | C++20 |
| **Toolchain** | LLVM / Clang 20 |
| **Build** | CMake |
| **Output** | JSON, SARIF |
| **CI** | clang-format enforcement, SARIF policy gates |

## License

Most repositories are licensed under **Apache 2.0** — permissive, patent-safe, and compatible with the LLVM toolchain. Standalone libraries (coretrace-log) use the **MIT** license for maximum integration flexibility.

## Contributing

Contributions are welcome through issues and pull requests on each repository.

<div align="center">

[Browse all repositories](https://github.com/orgs/CoreTrace/repositories) · [Website](https://github.com/CoreTrace/coretrace-website)

</div>
