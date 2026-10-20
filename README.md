# Paired-end-WGS-alignment-pipeline
Python scripts that run a short-read whole-genome workflow one step at a time, locally or as SLURM job arrays: SRA download, `bwa mem` alignment, then `samtools` fixmate / sort / index.


| Script | Covers |
|---|---|
| `01_get_reads.py` | `fastq-dump --split-files` download, paired-FASTQ check |
| `02_align_sort.py` | `bwa index` -> `bwa mem` -> `samtools fixmate` -> `sort` -> `index` |
| `03_postprocess.py` | optional: `samtools flagstat`, `samtools markdup` |

`common.py` holds the shared helpers and must stay in the same folder. All paths, sample names, accessions and SLURM
settings live in one JSON config, so the code contains nothing specific to a project.

```
SRA accession --fastq-dump--> FASTQ pair --bwa mem--> SAM --fixmate/sort/index--> sorted BAM (+bai) --> [flagstat, markdup]
```

## 1. Requirements

- Python >= 3.8 (standard library only)
- Tools on `PATH`: `bwa`, `samtools`, `fastq-dump` (sra-tools)
- Optionally SLURM (`sbatch`) for the `submit` command

```bash
conda env create -f environment.yml
conda activate wgs
```
(or load them with environment modules via `slurm.setup_commands`).

You also need a reference genome FASTA. Use the same FASTA (same contig names) for everything downstream.

## 2. Configure

Copy `config.example.json` to `config.json` and fill in every `Path_to_...` value.

| Key | Meaning |
|---|---|
| `work_dir` | All outputs are written here (`fastq/`, `sam/`, `bam/`, `stats/`) |
| `reference_fasta` | Reference FASTA; `bwa index` files must sit next to it |
| `threads` | `bwa mem -t` (default 30 on a 32-core job) |
| `samtools_threads` | `-@` for samtools; 1 (default) = no flag, as in the original runs |
| `read_group` | Optional `bwa mem -R` string, `{sample}` is replaced by the sample name, e.g. `"@RG\\tID:{sample}\\tSM:{sample}\\tPL:ILLUMINA"`. `null` = none |
| `fixmate_mate_score` | `true` adds `-m` to `samtools fixmate` (required if you will run `markdup`); default `false` = original behaviour |
| `scratch_dir` | Optional per-task temp dir (large disk) |
| `slurm` | partition, cpus, mem, time, array concurrency, and either `setup_commands` (shell/module lines) or `conda_sh` + `conda_env` |
| `samples` | one entry per sample (below) |

Sample entries: `{"name": "sample_1", "accession": "SRRXXXXXXX"}` (downloaded by step 1), or
`{"name": "sample_2", "fastq_1": "...", "fastq_2": "..."}` for FASTQ you already have. `"keep": false` skips a sample.
Downloaded files are named `fastq/<accession>_1.fastq` / `_2.fastq` by `fastq-dump`.

Keep `config.json` out of version control if it holds real names, accessions or paths.

## 3. Run

Add `--dry-run` before the subcommand to print commands without running. Per-sample steps take `--sample NAME`; under
SLURM they use the array task id instead.

```bash
P1="python 01_get_reads.py --config config.json"
P2="python 02_align_sort.py --config config.json"
P3="python 03_postprocess.py --config config.json"
```

| Step | Command | Underlying command | Output |
|---|---|---|---|
| download | `$P1 submit download` | `fastq-dump --split-files -O fastq ACC` | `fastq/ACC_1.fastq`, `_2.fastq` |
| check | `$P1 check` | line counts of both mates | pass/fail per sample |
| bwa index (once) | `$P2 index` | `bwa index REF` | `REF.{amb,ann,bwt,pac,sa}` |
| align | `$P2 submit align` | `bwa mem -t 30 REF fq1 fq2 > sam/S.sam` | `sam/<sample>.sam` |
| sort | `$P2 submit sort` | `samtools fixmate -O bam` -> `samtools sort -O bam -o` -> `samtools index` | `bam/<sample>_sorted.bam` + `.bai` |
| flagstat (opt.) | `$P3 submit flagstat` | `samtools flagstat` | `stats/<sample>.flagstat.txt` |
| markdup (opt.) | `$P3 submit markdup` | `samtools markdup -s` + `index` | `bam/<sample>_sorted.markdup.bam` |

Notes:
- `submit STEP` writes `slurm/STEP.sbatch` (one array task per kept sample, limited to `array_concurrency`), prints it and
  calls `sbatch`. Test one sample directly first (`$P2 align --sample sample_1`).
- Wait for an array to finish before submitting the next step; `sort` deletes the SAM and fixmate BAM when it succeeds
  (`--keep-sam` keeps them).
- Defaults (`-t 30`, 40 GB, 144 h, 32 CPUs, `ulimit -n 10000`) are the values of the original manual jobs.

## 4. Things that commonly go wrong

- **SAM files are huge.** A 30x human genome gives hundreds of GB of SAM. Make sure `work_dir` has the space; the sort step
  cleans up after itself.
- **bwa index missing or built for another FASTA.** `align` refuses to start without the `.bwt/.pac/.ann/.amb/.sa` files.
  Run `index` once (needs ~5 GB RAM and an hour or so for a human genome).
- **markdup after a plain fixmate.** `samtools markdup` needs the mate-score tags from `fixmate -m`; set
  `fixmate_mate_score` before the sort step if you plan to mark duplicates.
- **No read group, no duplicate marking by default.** The original runs used plain `bwa mem` with no `-R`. Most
  variant callers (GATK etc.) require a read group; set `read_group` before step 2 rather than patching the BAM later.
- **fixmate needs name-grouped input.** `bwa mem` output is name-grouped, so fixmate directly on the SAM is correct.
  Do not coordinate-sort before `fixmate`.
- **Open-file limit.** Sorting large BAMs can fail with "too many open files"; the generated sbatch script sets
  `ulimit -n 10000`.
- **Partial output.** If `bwa mem` fails, the partial SAM is deleted so a later step cannot pick it up.
- **FASTQ names.** `fastq-dump --split-files` yields `_1`/`_2`; reads that are not paired go to a third file that is
  ignored here. Run `check` before aligning.
