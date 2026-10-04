# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

IMPORTANT: Read [AGENTS.md](AGENTS.md) in full before beginning any work. It defines this project's strict AI usage policy. Key non-negotiable points:

- NEVER write PR descriptions, commit messages, reviewer responses, or GitHub comments on the user's behalf. NEVER run `git push`, `gh pr create`, `gh pr comment`, or `gh issue create`.
- Do not commit without explicit human approval for each commit. If explicitly asked to commit, use `Assisted-by: <assistant name>` (never `Co-authored-by:`).
- Search existing issues/PRs (`gh search issues`, `gh search prs`) before implementing anything.
- Prefer minimal assistance; the contributor must fully understand every change.

## Build

CMake-based. The primary product is the `llama` library (`include/llama.h`).

```bash
cmake -B build
cmake --build build --config Release -j 8

# Debug build (single-config generators)
cmake -B build -DCMAKE_BUILD_TYPE=Debug
```

On macOS, Metal and Accelerate are enabled by default. Backend-specific builds (CUDA, Vulkan, SYCL, etc.): see [docs/build.md](docs/build.md).

## Tests

```bash
# C++ tests (built from tests/, run via ctest)
ctest --test-dir build --output-on-failure

# Run a single test binary directly
./build/bin/test-sampling

# Or filter by name via ctest
ctest --test-dir build -R test-tokenizer -V
```

- `test-backend-ops` (in `tests/`) checks consistency of ggml operators across backends; required when modifying or adding ggml ops.
- Server tests are pytest-based: `cd tools/server/tests && ./tests.sh` (see its README for running individual tests).
- Full local CI: `bash ./ci/run.sh ./tmp/results ./tmp/mnt` (see [ci/README.md](ci/README.md)).
- Verify perplexity/performance are unaffected with `llama-perplexity` and `llama-bench`.

## Architecture

Four main layers, top to bottom:

- **`tools/` and `examples/`** - executables (`llama-cli` in `tools/cli`, `llama-server` in `tools/server`, `llama-bench`, `llama-quantize`, `llama-perplexity`, multimodal in `tools/mtmd`, ...).
- **`common/`** - shared helper code for the tools: argument parsing, sampling helpers, chat templates and the Jinja engine (`common/jinja/`), grammar/JSON-schema conversion.
- **`src/`** - the `llama` library implementation behind the C API in `include/llama.h`. Roughly one concern per file pair: `llama-model.cpp` (loading + per-architecture graph construction), `llama-arch.cpp` (architecture/tensor name tables), `llama-graph.cpp` (graph building blocks), `llama-context.cpp` (inference state), `llama-batch.cpp`, `llama-kv-cache*.cpp` and `llama-memory-*.cpp` (KV cache variants: unified, iSWA, recurrent, hybrid), `llama-vocab.cpp`, `llama-sampling.cpp`, `llama-grammar.cpp`.
- **`ggml/`** - the tensor library that executes computation graphs. Backends live in `ggml/src/ggml-<backend>/` (cpu, cuda, metal, vulkan, sycl, opencl, cann, ...). Device-agnostic scheduling/allocation in `ggml-backend.cpp` / `ggml-alloc.c`. Quantization formats in `ggml-quants.c`.

Model conversion from HuggingFace to GGUF is Python: `convert_hf_to_gguf.py` plus the `gguf-py/` package. Adding a new model touches both the Python converter and `src/llama-arch.cpp` / `src/llama-model.cpp`; see [docs/development/HOWTO-add-model.md](docs/development/HOWTO-add-model.md).

Key ggml conventions (from CONTRIBUTING.md):
- Tensors are row-major; dimension 0 is columns, 1 is rows, 2 is matrices.
- `C = ggml_mul_mat(ctx, A, B)` means $C^T = A B^T$.

## Coding style

- C++ with a C-style public API. Avoid third-party dependencies, fancy STL constructs, and templates; use basic `for` loops.
- 4-space indentation, brackets on same line, `void * ptr`, `int & a`; vertical alignment is valued.
- `snake_case` everywhere; name for longest common prefix (`number_small`, `number_big`).
- Sized integer types (`int32_t`) in the public API. `struct foo {}`, not typedefs; omit optional `struct`/`enum` keywords in C++ code.
- ASCII only: no emdash, unicode arrows, or `×`/`…` in code or comments.
- Keep comments minimal; do not add comments that restate the code (see examples in AGENTS.md).
- Follow surrounding code patterns; when in doubt, `clang-format` (clang-tools v15+).
