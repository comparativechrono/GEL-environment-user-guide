# The data

[Back to the guide](../README.md)

The simulator has the structure of two real Genomics England data releases, with every table and column of their data dictionaries, filled with **synthetic** rows. The rows were made for practising statistics: their proportions follow published 100,000 Genomes Project figures, but no row describes a real person, and a result you find tells you about the simulation, not about biology.

## The two data releases

| LabKeeper project | Data release | `folderPath` for R and Python | Participants | Tables |
| --- | --- | --- | --- | --- |
| main-programme | main-programme_v19_2024-10-31 | `/main-programme/main-programme_v19_2024-10-31` | 12,885: 4,000 rare-disease families (8,480 genomes) and 4,405 people with cancer | 127 |
| nhs-gms | nhs-gms-release_v5_2025-08-28 | `/nhs-gms/nhs-gms-release_v5_2025-08-28` | 1,850 NHS Genomic Medicine Service participants | 35 |

Use the main programme for most work. The NHS GMS release has the same kinds of data, organised by referral, and is useful for checking whether a finding holds in a second, independent group.

## What is in the tables

LabKeeper’s **Data Views** groups the tables by category. In the main programme:

| Category | Tables include | What they hold |
| --- | --- | --- |
| Quick View | `participant_summary`, `cancer_analysis`, `rare_disease_analysis`, `rare_disease_interpreted` | One-row-per-participant (or per analysis) summaries: the place to start |
| Common | `participant`, `clinic_sample`, `laboratory_sample`, `plated_sample`, `death_details` | Sex, year of birth, consent, the recruiting centre, family role; samples |
| Bioinformatics | `genome_file_paths_and_types`, `sequencing_report`, `aggregate_gvcf_sample_stats`, `tiering_data`, `tiered_variants_frequency`, `exomiser`, `panels_applied`, `gmc_exit_questionnaire`, `denovo_cohort_information`, `denovo_flagged_variants`, `cancer_tier_and_domain_variants`, `cancer_staging_consolidated` | Genome files and their quality; inferred ancestry and principal components; the variants found and how they were prioritised; the clinical teams’ conclusions; de novo variants; somatic and germline variants in cancer |
| Rare Disease | `rare_diseases_family`, `rare_diseases_pedigree`, `rare_diseases_participant_disease`, `rare_diseases_participant_phenotype`, `rare_diseases_gen_measurement`, `rare_diseases_imaging`, `rare_diseases_invest_blood_laboratory_test_report` … | Families and pedigrees; the disease each participant was recruited with; HPO terms; measurements, imaging and blood tests |
| Cancer | `cancer_participant_tumour`, `cancer_participant_disease`, `cancer_risk_factor_general`, `cancer_surgery`, `cancer_systemic_anti_cancer_therapy`, `pathology_reports` … | Tumours, pathology, risk factors, surgery and treatment |
| Secondary Data NHSE | `hes_apc`, `hes_op`, `hes_ae`, `hes_cc`, `ecds`, `did`, `mortality`, `cancer_register_nhsd`, `covid_test_results` | Hospital Episode Statistics (admissions, outpatients, A&E, critical care), emergency care, diagnostic imaging, deaths, cancer registrations, COVID-19 tests |
| NCRAS | `av_tumour`, `av_patient`, `av_treatment`, `av_rtd`, `av_imd`, `cwt`, `rtds`, `sact`, `ncras_did`, `lucada_2013`, `lucada_2014` | The national cancer registry: tumours and their stage, treatments, route to diagnosis, deprivation, waiting times, radiotherapy, chemotherapy, imaging, the lung cancer audit |
| Mental Health | 46 `mhsds_`, `mhldds_` and `mhmd_v4_` tables | Mental health services: referrals, contacts, diagnoses and assessments |
| Long Read Sequencing, Transcriptomics | `lrs_sequencing_data`, `rare_disease_ont_cohorts`, `rnaseq_qc_metrics` … | Small pilot cohorts and their files |
| Research Community provided data | `genomicsplc_prs_values`, `participant_supplementary_table` | Polygenic scores |

The NHS GMS release is organised by referral (`referral`, `referral_participant`, `referral_test`), with `condition` and `observation` for the clinical picture, and the same kinds of variant, cancer, registry and hospital tables.

**Columns and their codes.** Open a table and use **Schema Browser** for its columns, their types and coded values. The **About this release** panel links to the full data dictionary, which opens outside the simulator.

**Joining tables.** Almost every table has `participant_id`. Genomes join on the plate key (`platekey` or `plate_key`), and families on `rare_diseases_family_id`. Hospital diagnoses are ICD-10 codes written without the dot (`E119` for E11.9), and `hes_apc.diag_all` lists every diagnosis of an episode.

**First use of a table.** A table is loaded the first time something uses it, so the first query of a large table (such as `hes_apc`) takes a few seconds.

## The genotype files

The genotypes of every main-programme genome, at 7,200 common SNPs, are a PLINK file set, with principal components, inferred ancestry and relatedness alongside. Each folder has a `README.txt`: read it first.

| Path | What it holds |
| --- | --- |
| `/gel_data_resources/main_programme/aggregation/aggregate_gVCF_strelka/aggV2/additional_data/genotypes_plink/aggV2_common_variants_extract.bed`, `.bim`, `.fam` | Genotypes of the 12,885 main-programme genomes (GRCh38). The sample IDs are plate keys |
| `…/aggV2/additional_data/PCs_relatedness/aggV2_PCs_ancestry_relatedness.tsv` | Principal components, ancestry and relatedness for the same genomes |
| `…/aggV2/additional_data/PCs_relatedness/relatedness/` | KING-robust kinship between pairs of genomes |
| `/gel_data_resources/gms/aggregation/gms_germline_v5/additional_data/` | The same SNPs for the NHS GMS genomes, and their principal components |

In your home folder, `~/gel_data_resources` is a link to `/gel_data_resources`.

## Reading the data

### In LabKeeper

Open a table, filter it, and use **Export** to save a TSV, CSV or Excel file in `~/Downloads` (inside the RE), or **Export ▸ Script** for the R or Python code that fetches the same rows.

### From R

Rlabkey needs your username in a file called `~/.netrc`. Set it up once, in the terminal:

```bash
cd ~
cp -rf ~/gel_data_resources/example_config_files/Workspaces/. .
mousepad ~/.netrc      # replace yourusername with your username (e.g. jdoe), and the password with anything
chmod 600 ~/.netrc
```

Then, in RStudio:

```r
library(Rlabkey)
labkey.setDefaults(baseUrl = "https://labkey.prod.aws.gel.ac/labkey/")
v19 <- "/main-programme/main-programme_v19_2024-10-31"

# a table, or some of its columns and rows
participant <- labkey.selectRows(folderPath = v19, schemaName = "lists",
                                 queryName = "participant", colNameOpt = "rname",
                                 colSelect = c("participant_id", "participant_type", "year_of_birth"))

# or SQL, which can join tables
tumours <- labkey.executeSql(folderPath = v19, schemaName = "lists", colNameOpt = "rname",
                             sql = "SELECT participant_id, site_icd10_o2_3char FROM av_tumour")
```

More examples are in `~/gel_data_resources/example_scripts/labkey_api/`.

### From Python

```python
import pandas as pd
from labkey.api_wrapper import APIWrapper

api = APIWrapper(domain="labkey.prod.aws.gel.ac", container_path="/main-programme/main-programme_v19_2024-10-31",
                 context_path="labkey", use_ssl=True)
rows = api.query.execute_sql(sql="SELECT participant_id, year_of_birth FROM participant",
                             schema_name="lists", max_rows=100000000)
df = pd.DataFrame(rows["rows"])
```

### In the terminal

```bash
module load plink2/2.0.0-b.1-dev
G=/gel_data_resources/main_programme/aggregation/aggregate_gVCF_strelka/aggV2/additional_data/genotypes_plink/aggV2_common_variants_extract
plink2 --bfile $G --freq --out ~/allele_freq
```

Larger jobs go to the HPC: see [A closed environment](closed-environment.md#the-hpc).

## Other files

- `/genomes/by_date/…`: genome files (BAM, VCF). Find a participant’s files through `genome_file_paths_and_types`, never by browsing `/genomes`. In the simulator only the example genomes, the HCC1395 tumour and normal cell lines, have real files.
- `/gel_data_resources/example_scripts/`: example scripts, including workshops on relatedness, repeat expansions and BCFtools plugins, each with a README.
- `/pgen_public_data_resources/`: public reference data (reference genomes, ClinVar, gnomAD and others).
