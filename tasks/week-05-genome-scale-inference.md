# Week 5: genome-scale inference

[All tasks](README.md) · [Back to the guide](../README.md)

In this week’s [statistics practical](https://comparativechrono.github.io/statistics-training/week5.html) you saw what thousands of tests do to p-values, corrected for them with Bonferroni and the false discovery rate, met population structure and linkage disequilibrium, and ran a mini-GWAS of the FAMuSS SNPs. These tasks run a genome-wide association study (GWAS) of a hospital diagnosis on the Research Environment’s genotype files: quality control, a scan with and without principal components, corrections for the number of tests, QQ and Manhattan plots, and linkage disequilibrium around the top SNP. PLINK 2 does the scan in the Terminal, and R does everything else.

Several tasks touch questions from the [week 1 list](week-01-pre-registration.md#questions-you-could-ask). If one of them is your own project’s question, do it the way your plan says rather than the way the task says, and note anything you do differently in `changes.md`.

**Time:** about three hours. Parts marked *If you have time* are extra.\
**You will use:** the Terminal for PLINK 2, and RStudio.\
**You will hand in:** nothing this week, unless your course team asks. There is an optional Airlock export at the end, for practice.

## Before you start

1. Open the simulator, **RStudio** and a **Terminal**. Start `week5.qmd` in your project folder, `/re_gecip/training/<your username>`, with the same setup chunk as before (copy it from your `week4.qmd`), and run it.
2. In the Terminal, load PLINK 2 and name the genotype files:

    ```bash
    module load plink2/2.0.0-b.1-dev
    G=/gel_data_resources/main_programme/aggregation/aggregate_gVCF_strelka/aggV2/additional_data/genotypes_plink/aggV2_common_variants_extract
    ```

    The Terminal and R share your home folder (`~`), so R can read what PLINK writes, and the other way round. Each PLINK command here takes a minute or less.

**Two habits for many tests.** Decide everything you can before you see a p-value: the case definition, the covariates, the quality-control thresholds and the significance threshold. And report a finding with its effect size, its confidence interval and the number of tests behind it, not just its p-value.

## Task 1: cases, controls and covariates

*Who is in the study, and who has the disease?*

- **Tables and files:** `hes_apc`, `participant`, the ancestry file (see [The data](../docs/data.md#the-genotype-files))
- **Methods:** a case definition, covariates, principal components

1. **Ancestry.** Read the ancestry file, and plot the first two principal components, coloured by the inferred ancestry:

    ```r
    pcs <- read_tsv(file.path("/gel_data_resources/main_programme/aggregation/aggregate_gVCF_strelka/aggV2",
                              "additional_data/PCs_relatedness/aggV2_PCs_ancestry_relatedness.tsv"),
                    col_types = cols(participant_id = "c"))
    ggplot(pcs, aes(x = PC1, y = PC2, colour = ancestry0_8)) +
      geom_point(alpha = 0.3)
    ```

    Read it as a map of genetic similarity, as the practical says, not as a picture of separate groups.
2. **The cases.** This week’s example is type 2 diabetes: anyone with a hospital episode coded E11 (any of its subcodes, such as E119) anywhere in `diag_all`. Write your case definition in your report before you look at any results.

    ```r
    hes <- lk("hes_apc", c("participant_id", "diag_all"))
    cases <- hes %>%
      filter(str_detect(diag_all, "E11")) %>%
      distinct(participant_id) %>%
      pull(participant_id)
    ```

3. **The study and its files.** Keep the unrelated genomes, and give PLINK three files: the people to keep, the phenotype (PLINK codes controls 1 and cases 2) and the covariates (sex, age and the first 10 principal components):

    ```r
    people <- lk("participant", c("participant_id", "participant_phenotypic_sex", "year_of_birth"))
    study <- pcs %>%
      filter(unrelated_set == 1) %>%
      left_join(people, by = "participant_id") %>%
      mutate(T2D = if_else(participant_id %in% cases, 2, 1),
             sex = if_else(participant_phenotypic_sex == "Male", 1, 0),
             age = 2024 - year_of_birth)
    count(study, T2D)
    study %>% transmute(`#FID` = platekey, IID = platekey) %>% write_tsv("~/unrelated.txt")
    study %>% transmute(`#FID` = platekey, IID = platekey, T2D) %>% write_tsv("~/t2d.pheno")
    study %>%
      select(IID = platekey, sex, age, PC1:PC10) %>%
      mutate(`#FID` = IID, .before = 1) %>%
      write_tsv("~/covars.tsv")
    pcs %>%
      filter(ancestry0_8 == "EUR", unrelated_set == 1) %>%
      transmute(`#FID` = platekey, IID = platekey) %>%
      write_tsv("~/eur_unrelated.txt")
    ```

4. **Your threshold.** The genotype files hold a few thousand SNPs, not the millions of a real GWAS. Decide now, before you see any results, what p-value a SNP will need to count as significant in this scan, and write it in your report with your reasons.

**Think about:** who are your controls? Write down exactly who counts as a case and who as a control, and why.

## Task 2: quality control

*Remove the SNPs and samples you can’t trust, before you test anything.*

- **Files:** the genotype files, and your lists of people from task 1
- **Methods:** call rate, minor allele frequency (MAF) and Hardy–Weinberg equilibrium, as in the practical’s mini-GWAS

1. In the Terminal, measure missingness and allele frequencies in your study, and test Hardy–Weinberg equilibrium in a group of similar ancestry (as in week 3). This week, test the autosomes only:

    ```bash
    plink2 --bfile $G --autosome --keep ~/unrelated.txt --missing --freq --out ~/qc
    plink2 --bfile $G --autosome --keep ~/eur_unrelated.txt --hardy --out ~/qc_eur
    ```

2. In R, apply the usual thresholds of a GWAS: a call rate of at least 95% (at most 5% missing), a MAF of at least 1% (rarer SNPs have little power in a study of this size), and a Hardy–Weinberg p-value of at least 10<sup>−6</sup>:

    ```r
    qc <- read_tsv("~/qc.vmiss") %>%
      select(ID, F_MISS) %>%
      left_join(select(read_tsv("~/qc.afreq"), ID, ALT_FREQS), by = "ID") %>%
      left_join(select(read_tsv("~/qc_eur.hardy"), ID, HWE_P = P), by = "ID") %>%
      mutate(maf = pmin(ALT_FREQS, 1 - ALT_FREQS))
    qc %>% summarise(snps = n(), missing = sum(F_MISS > 0.05), rare = sum(maf < 0.01),
                     hwe = sum(HWE_P < 1e-6))
    fail <- qc %>% filter(F_MISS > 0.05 | maf < 0.01 | HWE_P < 1e-6)
    write_lines(fail$ID, "~/qc_fail.txt")
    ```

3. Remove the samples with more than 5% of their genotypes missing:

    ```r
    read_tsv("~/qc.smiss") %>%
      filter(F_MISS <= 0.05) %>%
      select(`#FID`, IID) %>%
      write_tsv("~/keep_qc.txt")
    ```

    Report how many SNPs each filter removed (a SNP can fail more than one), how many SNPs are left, and how many samples.

## Task 3: the scan, with and without principal components

*Test every SNP, with and without principal components, and compare the scans.*

- **Files:** your files from tasks 1 and 2
- **Methods:** logistic regression of each SNP, with covariates; QQ plots and the genomic inflation factor, λ

1. In the Terminal, run the scan twice: adjusted for sex and age only, and then for the first 10 principal components as well:

    ```bash
    plink2 --bfile $G --autosome --keep ~/keep_qc.txt --exclude ~/qc_fail.txt --pheno ~/t2d.pheno --covar ~/covars.tsv --covar-name sex,age --covar-variance-standardize --glm hide-covar firth-fallback --out ~/t2d_nopc
    plink2 --bfile $G --autosome --keep ~/keep_qc.txt --exclude ~/qc_fail.txt --pheno ~/t2d.pheno --covar ~/covars.tsv --covar-name sex,age,PC1-PC10 --covar-variance-standardize --glm hide-covar firth-fallback --out ~/t2d_pc
    ```

    Each writes a line for every SNP to a file ending `.T2D.glm.logistic.hybrid`: the allele tested (`A1`), the odds ratio for each copy of it (`OR`), its standard error on the log scale (`LOG(OR)_SE`) and the p-value (`P`).
2. In R, read both, and calculate λ for each, as in the practical:

    ```r
    res_nopc <- read_tsv("~/t2d_nopc.T2D.glm.logistic.hybrid")
    res_pc <- read_tsv("~/t2d_pc.T2D.glm.logistic.hybrid")
    lambda <- function(p) median(qchisq(p, df = 1, lower.tail = FALSE), na.rm = TRUE) / qchisq(0.5, df = 1)
    c(without_pcs = lambda(res_nopc$P), with_pcs = lambda(res_pc$P))
    ```

3. Draw the QQ plots of both scans on one graph:

    ```r
    qq <- bind_rows(without = res_nopc, with = res_pc, .id = "PCs") %>%
      filter(!is.na(P)) %>%
      group_by(PCs) %>%
      mutate(observed = -log10(sort(P)), expected = -log10(ppoints(n()))) %>%
      ungroup()
    ggplot(qq, aes(x = expected, y = observed, colour = PCs)) +
      geom_point() +
      geom_abline()
    ```

    What do λ and the QQ plots say? Which scan would you report, and why?

**Think about:** look at the SNPs that pass your threshold in each scan, if there are any. Which would you believe, and why?

## Task 4: many tests

*How many of your results could be chance, and which survive a correction?*

- **Files:** your `res_pc`
- **Methods:** the distribution of p-values, Bonferroni, Benjamini–Hochberg, genome-wide significance, Manhattan plots

1. **The p-values.** Draw a histogram of `res_pc$P` with a bar for each interval of 0.05 (`geom_histogram(breaks = seq(0, 1, by = 0.05))`). What shape is it, and what does the shape tell you?
2. **Corrections.** Count the p-values below 0.05 and compare them with the number you would expect by chance. Then apply both corrections, and count what passes each, and what passes genome-wide significance:

    ```r
    m <- sum(!is.na(res_pc$P))
    res_pc <- res_pc %>%
      mutate(bonferroni = p.adjust(P, method = "bonferroni"),
             bh = p.adjust(P, method = "BH"))
    c(tests = m, below_0.05 = sum(res_pc$P < 0.05, na.rm = TRUE), expected = m * 0.05,
      genome_wide = sum(res_pc$P < 5e-8, na.rm = TRUE),
      bonferroni = sum(res_pc$bonferroni < 0.05, na.rm = TRUE),
      bh = sum(res_pc$bh < 0.05, na.rm = TRUE))
    ```

3. **The top SNPs,** with a 95% confidence interval for each odds ratio:

    ```r
    top10 <- res_pc %>%
      arrange(P) %>%
      mutate(lower = exp(log(OR) - 1.96 * `LOG(OR)_SE`), upper = exp(log(OR) + 1.96 * `LOG(OR)_SE`)) %>%
      select(ID, A1, A1_FREQ, OR, lower, upper, P, bonferroni, bh) %>%
      head(10)
    top10
    ```

4. **A Manhattan plot,** with the SNPs in their places along the chromosomes, a dashed line at genome-wide significance and a dotted line at the Bonferroni threshold for this scan:

    ```r
    res_pc %>%
      mutate(chrom = factor(`#CHROM`, levels = 1:22)) %>%
      ggplot(aes(x = POS / 1e6, y = -log10(P))) +
      geom_point(size = 0.5) +
      geom_hline(yintercept = -log10(5e-8), linetype = "dashed", colour = "red") +
      geom_hline(yintercept = -log10(0.05 / m), linetype = "dotted") +
      facet_grid(. ~ chrom, scales = "free_x", space = "free_x") +
      theme(axis.text.x = element_blank(), axis.ticks.x = element_blank(),
            panel.spacing.x = unit(0, "lines")) +
      labs(x = "chromosome", y = "-log10(p)")
    ```

    Which SNPs pass the threshold you chose in task 1, and which pass each line on the plot?

**Think about:** if a SNP only just passes a threshold, what does the winner’s curse say about its odds ratio? What would convince you that it is real?

## Task 5: linkage disequilibrium

*Is your top SNP a point or a peak?*

- **Files:** the genotype files, your `res_pc`
- **Methods:** r² between SNPs, a regional association plot

1. In the Terminal, calculate r² between your top SNP and every SNP within 1,000 kb of it. LD differs between populations, so use the unrelated genomes of European ancestry. Replace `TOP_SNP` with your top SNP’s ID:

    ```bash
    plink2 --bfile $G --keep ~/eur_unrelated.txt --r2-unphased --ld-snp TOP_SNP --ld-window-kb 1000 --ld-window-r2 0 --out ~/ld_top
    ```

2. In R, plot the region around it, with each SNP coloured by its r² with the top SNP:

    ```r
    top <- res_pc %>% arrange(P) %>% slice(1)
    ld <- read_tsv("~/ld_top.vcor") %>% select(ID = ID_B, r2 = UNPHASED_R2)
    res_pc %>%
      filter(`#CHROM` == top$`#CHROM`, abs(POS - top$POS) < 1e6) %>%
      left_join(ld, by = "ID") %>%
      mutate(r2 = if_else(ID == top$ID, 1, r2)) %>%
      ggplot(aes(x = POS / 1e6, y = -log10(P), colour = r2)) +
      geom_point(size = 2) +
      scale_colour_gradient(low = "grey70", high = "firebrick") +
      labs(x = paste("position on chromosome", top$`#CHROM`, "(Mb)"))
    ```

    Do the SNPs in LD with your top SNP rise with it? Is the top SNP the cause of the association? What would it take to find out?
3. *If you have time:* run a scan of another diagnosis. Count its cases among the unrelated genomes first, then repeat task 1 (a new phenotype file), task 3 (a new `--out` name) and task 4. How does the number of cases change what a scan can find?

## Task 6: your own project

*How many tests does your plan make, and how will you allow for them?*

1. **Count your tests:** every outcome, exposure, subgroup and coding you will test. Does your plan name a correction (Bonferroni or Benjamini–Hochberg), and is it the right one for your question? If you change it, add a dated line to `changes.md`.
2. **If your project uses genotypes,** run the quality control of task 2 on your own study, include principal components as covariates, and check λ and the QQ plot.
3. **Replication.** Could your main finding be tested in an independent sample? The NHS GMS release has tables of the same kinds and its own genotype files.
4. **Write the inference section** of your report: the tests, the correction, what passed, and what you conclude. The practical’s homework uses the same structure.

## Your report

Put the analysis into `week5.qmd`: the case definition, the quality control (with the number each filter removed), λ and the QQ plots, the Manhattan plot, the top SNPs and the corrections, and an interpretation that considers population structure, LD, the size of the study and replication. Use inline code for the numbers, and `format.pval()` for p-values (see [week 3](week-03-testing-hypotheses.md#your-report)). The report reads PLINK’s files from your home folder, so run the PLINK steps first. Then **Render** it.

## Taking a result out (optional)

For practice, export your Manhattan plot (`ggsave("manhattan.png")` after drawing it) and your table of the top 10 SNPs (`write_csv(top10, "top10.csv")`) through the Airlock. Follow [the rules](../docs/airlock.md#what-may-leave), and export them under the training project **RR4001**.

**Think about:** why export the top 10 SNPs rather than the results for every SNP? (The Airlock’s rules say what happens to bulk variant lists.)
