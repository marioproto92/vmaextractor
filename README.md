# VMA Extractor with Progress & Multithreading

A fast and efficient Python tool to extract Proxmox VE `.vma` backups on Windows and Linux, featuring optional MD5 integrity checks, real-time progress feedback, and multithreaded I/O.

## Features

- **VMA Backup Extraction**: Parses the VMA header, metadata, and disk streams.
- **MD5 Validation (optional)**: Verify backup integrity; bypass with `--skip-hash`.
- **Progress Indicator**: Real-time updates every second to track extraction progress.
- **Multithreading**: Parallel disk cluster writes using `ThreadPoolExecutor`.
- **Cross-Platform**: Runs on Windows, Linux, and macOS with Python 3, no extra dependencies.

## Requirements

- Python 3.6 or later
- Standard Python modules: `hashlib`, `argparse`, `concurrent.futures`, `threading`

## Installation

1. Clone or download `vma_extractor.py` to your local machine.
2. Make the script executable (optional on Linux/macOS):
   ```bash
   chmod +x vma_extractor.py
   ```
3. Ensure Python 3 is installed and available in your `PATH`.

## Usage

```bash
./vma_extractor.py [OPTIONS] <backup.vma> <output_folder>
```

- `<backup.vma>`: The Proxmox VMA backup file (decompressed from `.zst`, e.g., via `zstd -d`).
- `<output_folder>`: Directory where configuration files and raw disk images will be saved.

### Options

| Flag              | Description                                             |
|-------------------|---------------------------------------------------------|
| `-f, --force`     | Overwrite the output folder if it already exists.       |
| `--skip-hash`     | Skip MD5 checksum validation for header and extents.    |

## Examples

- **Standard extraction with validation**:
  ```bash
  ./vma_extractor.py backup.vma extracted_vm
  ```
- **Force overwrite of existing folder**:
  ```bash
  ./vma_extractor.py -f backup.vma extracted_vm
  ```
- **Skip integrity checks**:
  ```bash
  ./vma_extractor.py --skip-hash backup.vma extracted_vm
  ```

## Performance

- **Multithreaded writes**: Utilizes all CPU cores for parallel I/O.
- **Fast-path optimization**: Bulk reads for full or empty clusters reduce overhead.
- **Minimal scripting overhead**: Progress updates throttled to once per second.

## Author

Mario Protopapa (Alias DeBuG)

## License

Released under the MIT License. Feel free to modify, enhance, and redistribute.
