# mlstdb2ariba

Convert [mlstdb](https://github.com/MDU-PHL/mlstdb) authenticated MLST database downloads to [ARIBA](https://github.com/sanger-pathogens/ariba)-compatible format.

## Background

`ariba pubmlstget` downloads MLST schemes from PubMLST without authentication. Therefore, it is limited to data pre-2025. For up-to-date, authenticated downloads, `mlstdb` can download the latest data, but it does not produce an ARIBA-compatible directory structure. This script fills that gap by converting `mlstdb` output into the format expected by ARIBA, allowing you to use the latest MLST data with ARIBA's powerful analysis tools without any manual reformatting.

## What it does

| Step | Details |
|---|---|
| Reads gene order | From `<scheme>.txt` profile header |
| Converts FASTA headers | `>arcC_1` → `>arcC.1` (underscore → dot) |
| Filters alleles | Removes alleles outside 90–110% of median length (mirrors ariba behaviour) |
| Writes `clusters.tsv` | One tab-separated row per gene, all allele names |
| Copies `profile.txt` | Into `pubmlst_download/` and `ref_db/` |
| Runs `ariba prepareref` | Builds the `ref_db/` ready for `ariba run` |

## Output structure

```
<outdir>/
  clusters.tsv                    ← one row per gene, tab-separated allele names
  pubmlst_download/
    arcC.tfa                      ← allele FASTAs with ariba-style headers (gene.N)
    aroE.tfa
    ...
    profile.txt
  ref_db/                         ← built by ariba prepareref
    pubmlst.profile.txt
    02.cdhit.clusters.tsv
    ...
```

This is identical to the output of `ariba pubmlstget`.

## Requirements

- Python ≥ 3.8 (standard library only — no extra Python packages needed)
- [ARIBA](https://github.com/sanger-pathogens/ariba) (must be on `$PATH` or specified with `--ariba`)
- [mlstdb](https://github.com/MDU-PHL/mlstdb) to produce the input directory

## Installation

```bash
git clone https://github.com/himal2007/mlstdb2ariba.git
cd mlstdb2ariba
chmod +x mlstdb_to_ariba.py
```

No dependencies to install. Uses Python standard library only.

## Usage

### Step 1: download the MLST database with mlstdb

```bash
mlstdb connect -d pubmlst          # register credentials (once only)
mlstdb update --input input.tab --directory mlstdb_custom/pubmlst --blast-directory mlstdb_custom/blast
```

Where `input.tab` contains:

```
database    species               scheme_description  scheme    URI
pubmlst     Staphylococcus aureus MLST                saureus   https://rest.pubmlst.org/db/pubmlst_saureus_seqdef/schemes/1
```

### Step 2: convert to ariba format

```bash
python mlstdb_to_ariba.py -i mlstdb_custom -o s_aureus_ariba
```

### Step 3: run ariba

```bash
ariba run s_aureus_ariba/ref_db reads_1.fq reads_2.fq output_dir
```

## Options

```
usage: mlstdb_to_ariba.py [-h] -i DIR -o DIR [-s NAME] [--ariba PATH] [--no-prepareref]

options:
  -h, --help            Show this help message and exit
  -i, --input DIR       mlstdb output directory (contains pubmlst/<scheme>/)
  -o, --output DIR      Output directory to create (must not exist)
  -s, --scheme NAME     Scheme name inside pubmlst/ (auto-detected if only one present)
  --ariba PATH          Path to ariba executable (default: 'ariba' on PATH)
  --no-prepareref       Skip ariba prepareref — only convert files and write clusters.tsv
```

## Examples

```bash
# Minimal — auto-detect scheme, run ariba prepareref
python mlstdb_to_ariba.py -i mlstdb_custom -o s_aureus_ariba

# Specify scheme explicitly (needed if mlstdb_dir has multiple schemes)
python mlstdb_to_ariba.py -i mlstdb_custom -s saureus -o s_aureus_ariba

# Convert files only, inspect before building the database
python mlstdb_to_ariba.py -i mlstdb_custom -o s_aureus_ariba --no-prepareref

# Use a specific ariba binary (e.g. inside a conda env)
python mlstdb_to_ariba.py -i mlstdb_custom -o s_aureus_ariba \
    --ariba ~/miniforge3/envs/ariba_env/bin/ariba
```

## Notes

- The `--no-prepareref` flag is useful for inspecting the converted files or for running `ariba prepareref` manually with custom options.
- Allele length filtering exactly mirrors ariba's internal behaviour (median ± 10%). A warning is printed for any filtered alleles.
- If your mlstdb directory contains multiple schemes, use `-s` to specify which one to convert.
- Issues and PRs are welcome! 

## License

This project is licensed under the GPLv3 License. See the [LICENSE](LICENSE) file for details.
