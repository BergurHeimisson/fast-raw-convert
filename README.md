# fastRawConvert

Fast multi-threaded CR3 RAW to JPEG batch converter, built on [LibRaw](https://www.libraw.org/) via the `rawpy` Python binding.

## Features

- Processes CR3 files in parallel across all CPU cores
- LibRaw releases the GIL during decoding, so threads run truly concurrently
- Output lands in a `jpeg/` subfolder inside the input directory
- Per-file timing and a final throughput summary
- Tunable quality, worker count, demosaicing algorithm, and resolution

## Requirements

| Dependency | Install |
|---|---|
| libraw | `brew install libraw` |
| rawpy | `pip3 install rawpy --break-system-packages` |
| Pillow | `pip3 install pillow --break-system-packages` |

Python 3.12 (Homebrew) is required. The shebang in `convert_raw.py` points to `/opt/homebrew/bin/python3.12`.

## Usage

```bash
# Basic — quality 90, all CPU cores
./convert_raw.sh /path/to/raw_photos

# Speed-first — half resolution + linear demosaicing
./convert_raw.sh /path/to/raw_photos --half-size --linear

# Custom quality and thread count
./convert_raw.sh /path/to/raw_photos --quality 85 --workers 8

# Recurse into subdirectories
./convert_raw.sh /path/to/raw_photos --recursive
```

Output is always written to `<input_dir>/jpeg/`.

## Options

| Flag | Default | Description |
|---|---|---|
| `--quality INT` | 90 | JPEG quality (1–95) |
| `--workers INT` | CPU count | Number of parallel threads |
| `--half-size` | off | Decode at half resolution (~2× faster) |
| `--linear` | off | Linear demosaicing (fastest, slightly softer) |
| `--recursive` | off | Search subdirectories for CR3 files |

## Speed tips

- `--linear` alone is the biggest single speed win with minimal visible quality loss at screen sizes.
- `--half-size` is ideal for proofing or thumbnail generation.
- Both flags together give maximum throughput.
- Worker count beyond CPU core count rarely helps since the bottleneck is CPU-bound decoding.
