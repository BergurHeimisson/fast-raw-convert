# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Setup

```bash
brew install libraw
pip3 install -r requirements.txt --break-system-packages
```

The scripts use the Homebrew Python shebang (`/opt/homebrew/bin/python3.12`).

## Running

```bash
# Basic conversion
./convert_raw.sh /path/to/raw_photos

# With options (fastest mode)
./convert_raw.sh /path/to/raw_photos --half-size --linear --workers 8

# Recursive, custom quality
./convert_raw.sh /path/to/raw_photos --recursive --quality 85
```

`convert_raw.sh` is a thin wrapper that resolves its own directory and execs `convert_raw.py` with all arguments forwarded.

## Architecture

Two files, no build system:

- **`convert_raw.py`** — all logic: CLI parsing, file discovery, conversion, progress reporting
- **`convert_raw.sh`** — 6-line bash wrapper for convenient invocation

### Data flow

```
input_dir → find_cr3_files() → ThreadPoolExecutor → convert_one() per file → JPEG output
```

### Concurrency model

Uses `ThreadPoolExecutor` (not `ProcessPoolExecutor`) because `rawpy`/LibRaw releases the Python GIL during C-level decode and postprocess operations, enabling true parallel execution on OS threads without inter-process overhead.

### Key functions in `convert_raw.py`

| Function | Purpose |
|----------|---------|
| `find_cr3_files(input_dir, recursive)` | Globs `*.CR3`/`*.cr3` |
| `convert_one(src, jpeg_root, quality, half_size, use_linear)` | Worker: opens CR3 with `rawpy`, calls `postprocess()`, saves JPEG via Pillow |
| `output_path(src, jpeg_root)` | Maps source path → output JPEG path |
| `main()` | CLI args, thread pool, progress output |

### CLI options

| Flag | Default | Effect |
|------|---------|--------|
| `--quality` | 90 | JPEG quality (1–95) |
| `--workers` | CPU count | Parallel threads |
| `--half-size` | off | Half-resolution decode (~2× faster) |
| `--linear` | off | Linear demosaicing (fastest, slightly softer) |
| `--recursive` | off | Search subdirectories |

## No tests exist

There is no test suite. Manual testing is done by running conversions on sample CR3 files.
