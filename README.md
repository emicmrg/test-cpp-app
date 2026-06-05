# ultracode MVP

`ultracode` is an ultralight local-first coding assistant MVP written in C++.

The goal is to test a practical thesis: a coding assistant for limited hardware should keep the CLI small, cache everything it can, retrieve only the relevant code, and let Ollama handle model execution.

## What this MVP does

- Scans a local repository.
- Builds structural-ish chunks for C/C++, JavaScript/TypeScript, Python, Markdown, and config files.
- Stores a local `.ultracode/` index with chunk files, vectors, and a TSV manifest.
- Calls Ollama `/api/embed` for embeddings when available.
- Falls back to a local hashed embedding when Ollama is not running.
- Supports hybrid retrieval: vector score + lexical score.
- Sends compact XML-tagged context to Ollama `/api/chat`.
- Prints file paths and line ranges as sources.

This first version intentionally avoids external C++ dependencies. It shells out to `curl` for Ollama calls so it can build with a plain C++20 compiler.

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

Run from the repo you want to analyze:

```bash
./build/ultracode init
./build/ultracode index
./build/ultracode stats
./build/ultracode search "authentication token"
./build/ultracode ask "Where is authentication handled?"
./build/ultracode explain src/main.cpp
```

## Commands

### `init`

Creates `.ultracode/config.json` and local index directories.

### `index`

Scans the current working directory, extracts chunks, generates vectors, and writes the local index.

Ignored directories include `.git`, `.ultracode`, `build`, `dist`, `target`, `node_modules`, virtualenv folders, and common editor folders.

### `search <query>`

Runs hybrid retrieval and prints the top chunks with scores.

### `ask <question>`

Retrieves relevant chunks, builds compact context, sends it to Ollama, and prints the answer plus sources.

### `explain <file>`

Sends a single file to the configured chat model for explanation.

## Current limitations

This MVP uses lightweight heuristic chunking instead of Tree-sitter. That is intentional for the first commit because the repo starts empty and the priority is to prove the full local flow end-to-end first.

Planned next step:

- Add a `ChunkExtractor` interface.
- Keep the heuristic extractor as fallback.
- Add a Tree-sitter-backed extractor for C/C++ first.
- Add incremental indexing by file hash.
- Replace shell-based curl calls with a small HTTP client abstraction.
- Add streaming chat output.
- Add a proper test suite.

## Architecture

```text
ultracode
├── repo scanner
├── heuristic chunk extractor
├── local index under .ultracode/
├── Ollama embedding client
├── local hashed embedding fallback
├── brute-force vector search
├── lexical search
├── hybrid ranker
└── Ollama chat client
```

## Why this shape

The CLI should stay cheap. On constrained machines, the LLM runtime is already the expensive part. This tool tries to preserve RAM/VRAM by caching embeddings, retrieving only a few chunks, and keeping context compact.
