# Variant-Calling Pipeline

A germline variant-calling pipeline that takes paired-end FASTQ files for a small cohort through QC, alignment to GRCh38, per-sample GVCF calling, joint genotyping, and filtering, ending with a filtered cohort VCF, a MultiQC report, and a machine-readable manifest.

The same pipeline is implemented and run several ways: plain Bash, Slurm job arrays on an HPC cluster, Docker/Apptainer containers, Snakemake, Nextflow, and AWS Batch. CI is set up through GitHub Actions.

## Pipeline stages

| # | Stage | What it does | Tools |
|---|-------|--------------|-------|
| 0 | validate | Checks the samplesheet and inputs (paired files, mate counts, truncated `.fastq.gz`) before any compute runs | Bash |
| 1 | qc_raw | QC on raw reads | FastQC |
| 2 | trim | Adapter and quality trimming | fastp |
| 3 | align | Alignment to full GRCh38 | BWA-MEM |
| 4 | postprocess | Sort, mark duplicates, index | samtools sort, GATK MarkDuplicates, samtools index |
| 5 | quantify | Per-sample variant calling in GVCF mode over a target region | GATK HaplotypeCaller |
| 6 | merge | Cohort joint genotyping | GATK GenomicsDBImport + GenotypeGVCFs |
| 7 | analyze | Hard-filtering of variants | GATK VariantFiltration |
| 8 | qc_report | Cohort-wide QC summary | MultiQC |
| 9 | publish | Tidy output tables + `manifest.json` | Bash |

Samples are defined in `samplesheet.csv`. No sample names are hardcoded in the code.

## One pipeline, several ways to run it

| Approach | Where | Key files |
|----------|-------|-----------|
| Bash | Local / single node | `run_pipeline.sh`, `run_sample.sh`, `run_cohort.sh`, `lib/` |
| Slurm job arrays | Northeastern Explorer HPC | `slurm/` |
| Containers | Docker / Apptainer | `containers/`, `IMAGE.md` |
| Snakemake | Local or cluster | `Snakefile`, `rules/`, `profiles/` |
| Nextflow | Local, Slurm, or AWS Batch | `main.nf`, `modules/`, `nextflow.config`, `conf/` |
| AWS | S3, ECR, AWS Batch on Fargate | `aws/` |

## Testing and CI

- **Smoke runs** on a small input set for each implementation (`smoke-run*/`).
- **Acceptance tests** in `tests/`.
- **GitHub Actions** (`.github/workflows/`) runs ShellCheck and a full smoke-test pipeline run on every push and pull request.

## Troubleshooting log

`TROUBLESHOOTING.md` records real failures hit while building and scaling the pipeline, each with the symptom, evidence, cause, and fix. Examples include:

- A GenomicsDBImport workspace written to NFS storage instead of node-local `/tmp`, causing hour-long timeouts on the cluster.
- A reference passed as a Nextflow queue channel instead of a value channel, which silently dropped samples with no error.
- An AWS Batch job failing because the output paths it assumed didn't match the pipeline's actual output layout (diagnosed through CloudWatch logs).
- A Nextflow `.dict` path losing its `s3://` prefix when run on AWS Batch.

## How to run

```bash
# Bash (laptop / single node): all ten stages
./run_pipeline.sh samplesheet.csv out

# Stop after a given stage, e.g. validation only
./run_pipeline.sh samplesheet.csv out validate

# Smoke run on a small reference/region
REF=$PWD/smoke.fa REGION=smoke_1mb bash run_pipeline.sh samplesheet.csv out

# Nextflow: local with Docker
nextflow run main.nf -profile docker

# Nextflow: Northeastern Explorer HPC (Slurm + Apptainer)
nextflow run main.nf -profile explorer

# Nextflow: AWS Batch
nextflow run main.nf -profile awsbatch

# Snakemake (see profiles/ for cluster settings)
snakemake --cores 4
```

## Resources

`RESOURCES.md` lists the reference genome and other inputs the pipeline needs.

---

*Built for BINF6610 (Scientific Computing for Bioinformatics), Northeastern University.*
