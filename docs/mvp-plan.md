# MVP Plan

## MVP goal

Build a local-first terminal coding assistant that can run acceptably on limited hardware by using a small C++ CLI, local indexing, cached embeddings, compact retrieval, and Ollama for model execution.

## Success criteria

- Build with CMake and a C++20 compiler.
- Work without internet after dependencies/models are available.
- Index a small or medium repository without crashing.
- Avoid re-sending the whole repository to the model.
- Return source paths and line ranges.
- Keep the CLI lightweight; the model process remains external.

## Phase 1: end-to-end local flow

Implemented in this MVP:

- CLI commands: `init`, `index`, `stats`, `search`, `ask`, `explain`.
- Local `.ultracode/` workspace.
- Heuristic code chunking for C/C++, Go, Python, JavaScript/TypeScript, Markdown, and config files.
- Go chunking for functions, methods with receivers, structs, and interfaces.
- Ollama `/api/embed` integration.
- Ollama `/api/chat` integration.
- Local fallback embeddings.
- Brute-force vector search.
- Lexical scoring.
- Hybrid ranking.

## Phase 1.5: project structure refactor

Before adding Tree-sitter or more advanced retrieval, split the current single-file prototype into small, testable modules.

Goals:

- Keep `main.cpp` as a thin CLI entry point only.
- Move configuration loading into `config.hpp/.cpp`.
- Move filesystem scanning and ignore handling into `repo_scanner.hpp/.cpp`.
- Move chunk models and extraction logic into `chunk.hpp`, `chunk_extractor.hpp/.cpp`, and language-specific heuristic extractors.
- Move Ollama calls into `ollama_client.hpp/.cpp`.
- Move vector persistence and manifest handling into `index_store.hpp/.cpp`.
- Move ranking and retrieval into `retriever.hpp/.cpp`.
- Add a small test fixture repo to validate chunking and retrieval behavior.

Exit criteria:

- `main.cpp` is under roughly 100 lines.
- The project still builds with plain CMake and C++20.
- `init`, `index`, `stats`, `search`, `ask`, and `explain` keep the same CLI behavior.
- Go, C/C++, Python, JS/TS, Markdown, and config indexing still work after the split.

## Phase 2: better parsing

- Introduce `ChunkExtractor` interface.
- Add Tree-sitter C/C++ extractor.
- Add Go, Python, and TypeScript Tree-sitter extractors.
- Preserve heuristic extractor as fallback.

## Phase 3: better performance

- Incremental indexing by file hash.
- Batched `/api/embed` calls.
- Memory-mapped vector store or compact binary format.
- Optional HNSW/USearch backend for large repos.

## Phase 4: coding assistant features

- Streaming chat output.
- Patch generation.
- Git diff awareness.
- `/context`, `/model`, `/clear` interactive commands.
- Safe apply/reject workflow for edits.
