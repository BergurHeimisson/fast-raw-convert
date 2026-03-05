# Architecture

## Overview

```
convert_raw.sh          <- thin bash wrapper, resolves script dir, execs python
convert_raw.py          <- all logic lives here
```

## Data flow

```
input_dir/
  ├── IMG_0001.CR3
  ├── IMG_0002.CR3
  └── ...
        │
        │  find_cr3_files()
        ▼
  [Path, Path, ...]
        │
        │  ThreadPoolExecutor  (N workers, default = CPU count)
        │
        ├── convert_one(IMG_0001.CR3) ──► rawpy.imread ──► raw.postprocess ──► PIL.save
        ├── convert_one(IMG_0002.CR3) ──► rawpy.imread ──► raw.postprocess ──► PIL.save
        └── ...
        │
        ▼
  input_dir/jpeg/
    ├── IMG_0001.jpg
    ├── IMG_0002.jpg
    └── ...
```

## Key functions

### `find_cr3_files(input_dir, recursive) -> list[Path]`
Globs for `*.CR3` / `*.cr3` in the input directory. With `--recursive` uses `rglob` to descend into subdirectories.

### `convert_one(src, jpeg_root, quality, half_size, use_linear) -> (Path, float, str | None)`
The per-file worker. Called on a thread-pool thread.

1. Opens the RAW file with `rawpy.imread` (LibRaw under the hood).
2. Calls `raw.postprocess(params)` to demosaic and produce an 8-bit RGB numpy array.
3. Wraps the array in a `PIL.Image` and saves as JPEG.
4. Returns the source path, elapsed wall time, and an error string (or `None` on success).

### `main()`
Parses CLI arguments, builds the thread pool, dispatches all files, and prints progress as futures complete via `as_completed`.

## Concurrency model

`ThreadPoolExecutor` is used rather than `ProcessPoolExecutor` because LibRaw releases the Python GIL during its C-level decode and postprocess calls. This means OS threads run truly in parallel without the overhead of inter-process communication or pickling.

Worker count is capped at `min(--workers, total_files)` to avoid spinning up idle threads.

## LibRaw parameters

| Parameter | Value | Reason |
|---|---|---|
| `use_camera_wb` | `True` | Uses the camera's recorded white balance; fast, no analysis pass |
| `no_auto_bright` | `False` | Allows LibRaw to apply mild auto-brightness |
| `output_bps` | `8` | 8-bit output required for standard JPEG |
| `half_size` | CLI flag | Skips full-resolution demosaicing; roughly 2× faster |
| `demosaic_algorithm` | `AHD` (default) / `LINEAR` (`--linear`) | AHD is high quality; LINEAR is the fastest available algorithm |

## JPEG encoding

Pillow is used for JPEG encoding with:
- `optimize=False` — skips the entropy optimization pass for speed
- `subsampling=2` — 4:2:0 chroma subsampling, standard for photographic content

## Output layout

Output files are always written flat into `<input_dir>/jpeg/`, regardless of any subdirectory structure in the input. File names preserve the original stem with a `.jpg` extension.
