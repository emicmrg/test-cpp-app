# ultracode MVP

`ultracode` is an ultralight local-first coding assistant MVP written in C++.

The goal is to test a practical thesis: a coding assistant for limited hardware should keep the CLI small, cache everything it can, retrieve only the relevant code, and let Ollama handle model execution.

## What this MVP does

- Scans a local repository.
- Builds structural-ish chunks for C/C++, Go, JavaScript/TypeScript, Python, Markdown, and config files.
- Stores a local `.ultracode/` index with chunk files, a binary vector store, manifests, and per-file indexing state.
- Reindexes incrementally by file content hash instead of recomputing the entire repo every run.
- Calls Ollama `/api/embed` with batched requests when available.
- Falls back to a local hashed embedding when Ollama is not running.
- Supports hybrid retrieval: vector score + lexical score.
- Prioritizes changed files when the working tree has a local `git diff`.
- Sends compact XML-tagged context to Ollama `/api/chat`.
- Supports interactive chat with session commands.
- Generates unified diff patch proposals and supports explicit `apply` / `reject` flows.
- Prints file paths and line ranges as sources.

This first version intentionally avoids external C++ dependencies. It shells out to `curl` for Ollama calls so it can build with a plain C++20 compiler.

## Parsing support

The indexer supports these languages:

- Heuristic chunking for C/C++, Go, JavaScript/TypeScript, Python, Markdown, and config files.
- Optional Tree-sitter-backed extraction for C/C++, Go, Python, and TypeScript/JavaScript.

Heuristic extraction covers:

- C/C++: classes, structs, enums, functions.
- Go: functions, methods with receivers, structs, and interfaces.
- Python: classes and functions.
- JavaScript/TypeScript: class/function-like blocks using the C-like extractor.
- Markdown: sections by heading.
- Config files: line-based chunks.

## Requirements

- C++20 compiler
- CMake 3.20+
- `curl`
- Optional but recommended: Ollama running locally

Recommended Ollama models for limited hardware:

```bash
ollama pull nomic-embed-text
ollama pull qwen2.5-coder:3b
```

If `qwen2.5-coder:3b` is too heavy, edit `.ultracode/config.json` and use `qwen2.5-coder:1.5b`.

## Build

```bash
cmake -S . -B build
cmake --build build
```

Tree-sitter is optional and disabled by default. To enable it:

```bash
cmake -S . -B build -DULTRACODE_USE_TREESITTER=ON
cmake --build build
```

Run from the repo you want to analyze:

```bash
./build/ultracode init
./build/ultracode index
./build/ultracode stats
./build/ultracode search "authentication token"
./build/ultracode ask "Where is authentication handled?"
./build/ultracode chat
./build/ultracode patch "rename Greeter to WelcomeMessage"
./build/ultracode apply <patch-id>
./build/ultracode reject <patch-id>
./build/ultracode explain src/main.cpp
```

## Commands

### `init`

Creates `.ultracode/config.json` and local index directories.

### `index`

Scans the current working directory, extracts chunks, generates vectors, and writes the local index.

This command is incremental:

- reuses unchanged files by content hash
- reindexes only new or modified files
- removes chunk artifacts for deleted files
- updates a binary `vectors.bin` store plus per-file state under `.ultracode/`

Ignored directories include `.git`, `.ultracode`, `build`, `dist`, `target`, `node_modules`, virtualenv folders, and common editor folders.

### `search <query>`

Runs hybrid retrieval and prints the top chunks with scores. If the repo has local `git` changes, changed files receive a ranking boost.

### `ask <question>`

Retrieves relevant chunks, adds local diff context when available, sends the prompt to Ollama, and prints the answer plus sources.

### `chat`

Starts an interactive session backed by retrieval and streaming chat output.

Supported in-session commands:

- `/context`
- `/model <name>`
- `/clear`
- `/exit`

### `patch <instruction>`

Asks the model for a unified diff, stores it under `.ultracode/patches/`, and prints the generated patch id plus affected files.

### `apply <patch-id>`

Validates a stored patch with `git apply --check` and applies it only if the working tree has not drifted.

### `reject <patch-id>`

Marks a stored patch proposal as rejected without touching files.

### `explain <file>`

Sends a single file to the configured chat model for explanation.

## Current limitations

Current tradeoffs:

- HTTP access still shells out to `curl` instead of using an internal client.
- The vector index is still brute-force search; there is no ANN backend yet.
- Patch generation depends on the model returning a valid unified diff.
- Patch application uses `git apply`, but there is no interactive diff preview command yet.
- Tree-sitter support is optional and still not the default build path.

## Architecture

```text
ultracode
├── app/        command handlers and orchestration
├── cli/        CLI parsing and help
├── edit/       patch proposal persistence and apply/reject flow
├── index/      manifests, vectors, repo scan, incremental index state
├── llm/        Ollama embeddings and chat client
├── parsing/    chunk models, heuristic extraction, optional Tree-sitter
├── retrieval/  hybrid ranking and source selection
├── session/    interactive chat session state and commands
├── support/    shared utility helpers
└── vcs/        git diff context loading
```

## Why this shape

The CLI should stay cheap. On constrained machines, the LLM runtime is already the expensive part. This tool tries to preserve RAM/VRAM by caching embeddings, retrieving only a few chunks, and keeping context compact.
