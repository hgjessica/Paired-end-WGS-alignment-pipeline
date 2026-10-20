# Paired-end-WGS-alignment-pipeline
Python scripts that run a short-read whole-genome workflow one step at a time, locally or as SLURM job arrays: SRA download, `bwa mem` alignment, then `samtools` fixmate / sort / index.
