# Week 3: testing hypotheses and comparing groups

[All tasks](README.md) · [Back to the guide](../README.md)

In this week’s [statistics practical](https://comparativechrono.github.io/statistics-training/week3.html) you built a test by hand by shuffling labels, compared two groups and several groups, tested tables of counts and genotypes, and practised choosing a test. These tasks use the same tests on the Research Environment’s data. Work in RStudio in the simulator, and keep the tests that matter in a Quarto report, as you did in the practical.

Several tasks touch questions from the [week 1 list](week-01-pre-registration.md#questions-you-could-ask). If one of them is your own project’s question, do it the way your plan says rather than the way the task says, and note anything you do differently in `changes.md`.

**Time:** about three hours. Parts marked *If you have time* are extra.\
**You will use:** RStudio, LabKeeper’s Schema Browser, and the Terminal for task 5.\
**You will hand in:** nothing this week, unless your course team asks. There is an optional Airlock export at the end, for practice.

## Before you start

1. Open the simulator and **RStudio**. Your `~/.netrc` and the tidyverse from week 2 are still there. If you haven’t set up `~/.netrc`, see [The data](../docs/data.md#from-r); if R says there is no package called ‘tidyverse’, install it with `install.packages("tidyverse")`.
2. Start your report: **File ▸ New File ▸ Quarto Document…**, and save it as `week3.qmd` in your project folder, `/re_gecip/training/<your username>`. It needs the same setup chunk as last week’s: copy it from your `week2.qmd` (copy and paste work inside the RE), or type it:

    ````markdown
    ```{r}
    #| label: setup
    #| message: false
    library(tidyverse)
    library(Rlabkey)
    labkey.setDefaults(baseUrl = "https://labkey.prod.aws.gel.ac/labkey/")
    v19 <- "/main-programme/main-programme_v19_2024-10-31"

    # lk("table") fetches a table; lk("table", c("col1", "col2")) only those columns
    lk <- function(table, cols = NULL) {
      as_tibble(labkey.selectRows(folderPath = v19, schemaName = "lists", queryName = table,
                                  colSelect = cols, colNameOpt = "rname"))
    }
    ```
    ````

    Run it with **Run Current Chunk** (the ▶ at the chunk’s top right, or Ctrl+Shift+Enter).

**Two habits for tests.** Look before you test: plot and summarise the groups first, so that you know what the test compares and whether its assumptions are reasonable. And report the **estimate** (a difference, or an odds ratio) with its **95% confidence interval**, not just the p-value. Last week’s habits still apply: find out what one row of a table represents, and read the coded values in the Schema Browser.

## Task 1: the logic of a test

*Does the way a tumour sample was prepared change its measured tumour mutational burden? Build the test yourself, by shuffling labels.*

- **Tables:** `cancer_analysis`
- **Methods:** null and alternative hypotheses, a test statistic, a permutation test, the p-value

Tumour samples are prepared in one of two main ways: frozen fresh (`FF`), or fixed in formalin and embedded in paraffin (`FFPE`), which is how pathology laboratories usually keep tissue. The column `preparation_method` says which.

1. **The hypotheses.** Write the null and the alternative hypotheses in your report, in words, before you look at the data.
2. **Look first.** Fetch the data, count the samples prepared each way, keep the FF and FFPE samples, and compare the two groups. Work with the log of tumour mutational burden (`log10()`); task 2 checks whether that was a good choice.

    ```r
    cancer <- lk("cancer_analysis", c("participant_id", "disease_type", "study_abbreviation",
                                      "preparation_method", "somatic_coding_variants_per_mb")) %>%
      mutate(log_tmb = log10(somatic_coding_variants_per_mb))
    count(cancer, preparation_method)
    prep <- cancer %>% filter(preparation_method %in% c("FF", "FFPE"))
    prep %>%
      group_by(preparation_method) %>%
      summarise(n = n(), mean = mean(log_tmb), sd = sd(log_tmb))
    ggplot(prep, aes(x = preparation_method, y = log_tmb)) +
      geom_boxplot()
    ```

3. **The test statistic** is the difference between the two groups’ mean log TMB. Write it as a function, as in the practical, and calculate it:

    ```r
    diff_means <- function(y, group) {
      mean(y[group == "FFPE"]) - mean(y[group == "FF"])
    }
    obs <- diff_means(prep$log_tmb, prep$preparation_method)
    obs
    ```

    On a log10 scale, a difference of d means that one group’s values are typically 10<sup>d</sup> times the other’s, so `10^obs` turns your difference into a ratio.
4. **The null distribution.** Shuffle the labels 5,000 times, draw the null distribution with your observed difference on it, and work out the p-value:

    ```r
    set.seed(1)
    null <- replicate(5000, diff_means(prep$log_tmb, sample(prep$preparation_method)))
    tibble(null = null) %>%
      ggplot(aes(x = null)) +
      geom_histogram(bins = 50) +
      geom_vline(xintercept = obs, colour = "red")
    mean(abs(null) >= abs(obs))
    ```

    How precise is a p-value from 5,000 shuffles? What would you report if none of them, or very many, were as extreme as your difference?
5. *If you have time:* shuffling works for any statistic. Write `diff_medians()`, and test the difference between the medians of the raw tumour mutational burden in the same way (1,000 shuffles are enough).

**Think about:**

- Write your result as a sentence that begins “If the way a sample was prepared made no difference, …”.
- The samples weren’t allocated to FF or FFPE at random. What else could differ between the two groups of samples, and could it explain a difference?

## Task 2: two groups

*Compare two groups with a t-test and a Wilcoxon test, and paired measurements with a paired test.*

- **Tables:** `cancer_analysis` (your `prep`), `denovo_cohort_information`, `participant`, `gmc_exit_questionnaire`, `rare_diseases_imaging`, `rare_diseases_invest_blood_laboratory_test_report`
- **Methods:** Welch’s t-test, QQ plots, the Wilcoxon rank-sum test, the paired t-test

1. **A t-test.** Compare log TMB between the FF and FFPE samples:

    ```r
    t.test(log_tmb ~ preparation_method, data = prep)
    ```

    Read it from the bottom, as in the practical. R takes the groups in alphabetical order, so the difference is FF − FFPE, on the log10 scale: 10 to the power of the difference, and of each end of its interval, gives a ratio with its 95% confidence interval. How does the p-value compare with your permutation test’s?
2. **Assumptions.** Draw QQ plots of the raw TMB in each group, and then of `log_tmb`:

    ```r
    ggplot(prep, aes(sample = somatic_coding_variants_per_mb)) +
      stat_qq() +
      stat_qq_line() +
      facet_wrap(~ preparation_method)
    ```

    Which is closer to normal? Then compare the raw values with a Wilcoxon test, `wilcox.test(somatic_coding_variants_per_mb ~ preparation_method, data = prep)`. Would it give a different p-value if you used `log_tmb` instead? Why, or why not?
3. **Paired data.** Do fathers and mothers differ in age when their child is born? Each trio in `denovo_cohort_information` gives a pair of ages: the father’s and the mother’s at the child’s birth, from their years of birth in `participant`.

    ```r
    trios <- lk("denovo_cohort_information", c("trio_id", "family_id", "participant_id", "member"))
    born <- lk("participant", c("participant_id", "year_of_birth"))
    ages <- trios %>%
      left_join(born, by = "participant_id") %>%
      select(trio_id, family_id, member, year_of_birth) %>%
      pivot_wider(names_from = member, values_from = year_of_birth) %>%
      mutate(father_age = Offspring - Father, mother_age = Offspring - Mother)
    ```

    Look at the ages first. Then compare them with a paired t-test, `t.test(ages$father_age, ages$mother_age, paired = TRUE)`, and with a two-sample t-test (the same without `paired = TRUE`). Which gives the larger t, and why?
4. **Does the gene change the disease?** Is total kidney volume different in polycystic kidney disease caused by PKD1 and by PKD2? The diagnosed gene is in `gmc_exit_questionnaire`, and the volume in `rare_diseases_imaging`, where it is stored as text (the Schema Browser shows each column’s type), so `as.numeric()` turns it into numbers:

    ```r
    genes <- lk("gmc_exit_questionnaire", c("participant_id", "case_solved_family", "gene_name"))
    kidney <- lk("rare_diseases_imaging", c("participant_id", "total_kidney_volume")) %>%
      mutate(total_kidney_volume = as.numeric(total_kidney_volume))
    pkd <- genes %>%
      filter(gene_name %in% c("PKD1", "PKD2")) %>%
      left_join(kidney, by = "participant_id")
    ```

    Decide which probands to include, count them, look at the two distributions, and choose your test: a t-test of the raw or the logged volume, or a Wilcoxon test. Report the difference with its confidence interval.
5. *If you have time:* do the same for HbA1c (`hba1c_result` in `rare_diseases_invest_blood_laboratory_test_report`, also stored as text) in diabetes caused by GCK and by HNF1A. Which test did you choose, and why?

**Think about:** can a family appear more than once in `ages`? (Compare the numbers of trios and of families with `n_distinct()`.) If it can, are your pairs independent, and what could you do about it?

## Task 3: several groups

*Does tumour mutational burden differ between cancer types, and which types differ?*

- **Tables:** `cancer_analysis` (your `cancer` from task 1), and for the extra task `tiering_data` and `participant_summary`
- **Methods:** one-way ANOVA, Tukey’s test, the Kruskal–Wallis test

1. **Why not t-tests?** Keep the cancer types with at least 100 tumours:

    ```r
    common <- cancer %>% count(study_abbreviation) %>% filter(n >= 100) %>% pull(study_abbreviation)
    types <- cancer %>% filter(study_abbreviation %in% common)
    ```

    How many pairs of types are there? If you compared each pair with a t-test at α = 0.05, and no types really differed, what would be the chance of at least one false positive?
2. **ANOVA.** Summarise log TMB for each type (n, mean and SD) and draw box plots. Then do the ANOVA:

    ```r
    fit <- aov(log_tmb ~ study_abbreviation, data = types)
    summary(fit)
    ```

3. **Which types differ?** Tukey’s test compares every pair. Turn its result into a table, sorted by its adjusted p-values, and print all of it:

    ```r
    tukey <- as_tibble(TukeyHSD(fit)$study_abbreviation, rownames = "pair")
    tukey %>% arrange(`p adj`) %>% print(n = Inf)
    ```

    Which pairs differ, and which can’t be told apart?
4. **Assumptions.** ANOVA assumes that the values in each group are roughly normal, with a similar spread. Look at your SDs and box plots. Then compare the raw TMB with a Kruskal–Wallis test, `kruskal.test(somatic_coding_variants_per_mb ~ study_abbreviation, data = types)`.
5. *If you have time:* does the number of Tier 3 variants a proband has differ between ancestry groups? Count each rare-disease proband’s Tier 3 variants in `tiering_data` (keep the probands with none, as you did with HPO terms in week 2), take their ancestry from `genetically_inferred_ancestry_thr` in `participant_summary`, and compare the groups. Which test did you choose, and why?

## Task 4: counts and odds ratios

*Test tables of counts, and measure associations with odds ratios.*

- **Tables:** `gmc_exit_questionnaire`, `rare_diseases_participant_disease`, `cancer_tier_and_domain_variants`, `cancer_analysis`
- **Methods:** tables of counts, the chi-square test and expected counts, Fisher’s exact test, the odds ratio

1. **Diagnostic yield by disease group.** In week 2 you put a confidence interval on each disease group’s yield; now test whether the yield is the same in every group. Make `yield`, with one row per proband, a column `solved` (TRUE or FALSE, counted as you decided in week 2) and the proband’s `normalised_disease_group`, from these tables:

    ```r
    exitq <- lk("gmc_exit_questionnaire", c("participant_id", "case_solved_family"))
    dis <- lk("rare_diseases_participant_disease", c("participant_id", "normalised_disease_group"))
    ```

    Then test it:

    ```r
    tab <- table(yield$normalised_disease_group, yield$solved)
    ct <- chisq.test(tab)
    ct
    ct$expected
    ```

    Are all the expected counts at least 5? What does the test tell you, and what doesn’t it?
2. **One group against the rest.** Choose a disease group, and compare its yield with that of all the other groups together:

    ```r
    tab2 <- table(yield$normalised_disease_group == "the group you chose", yield$solved)
    tab2
    fisher.test(tab2)
    ```

    Write the odds ratio as a sentence: whose odds of what, compared with whose? (Look at the order of the rows and columns.)
3. **Two driver genes.** In colorectal cancer, are somatic mutations of KRAS and BRAF found in the same tumour more often, or less often, than chance would give?

    ```r
    variants <- lk("cancer_tier_and_domain_variants", c("participant_id", "origin", "gene"))
    somatic <- variants %>% filter(origin == "somatic")
    crc <- cancer %>%
      filter(disease_type == "COLORECTAL") %>%
      mutate(kras = participant_id %in% somatic$participant_id[somatic$gene == "KRAS"],
             braf = participant_id %in% somatic$participant_id[somatic$gene == "BRAF"])
    table(KRAS = crc$kras, BRAF = crc$braf)
    ```

    Test the table, and say what its odds ratio means.

**Think about:** if you chose your group in step 2 after seeing the yields in step 1, what does that do to its p-value? How many tests would it take to compare every group with the rest? (Week 5 is about this.)

## Task 5: Hardy–Weinberg equilibrium

*Test the SNPs in the genotype files for Hardy–Weinberg equilibrium, in all the genomes and then in one ancestry group.*

- **Files:** the genotype files and the ancestry file (see [The data](../docs/data.md#the-genotype-files))
- **Methods:** the Hardy–Weinberg test as a quality check; testing within a group of similar ancestry

1. In the **Terminal**, test every SNP in all the genomes:

    ```bash
    module load plink2/2.0.0-b.1-dev
    G=/gel_data_resources/main_programme/aggregation/aggregate_gVCF_strelka/aggV2/additional_data
    plink2 --bfile $G/genotypes_plink/aggV2_common_variants_extract --hardy --out ~/hwe_all
    ```

    PLINK writes a line for each autosomal SNP to `~/hwe_all.hardy`: its genotype counts, the observed and expected proportions of heterozygotes, and the p-value of an exact test of equilibrium. The SNPs on the X chromosome go to a second file, `~/hwe_all.hardy.x`, because men have only one X; leave them out this week.
2. In **R**, make a list of the unrelated genomes of European ancestry from the ancestry file:

    ```r
    pcs <- read_tsv(file.path("/gel_data_resources/main_programme/aggregation/aggregate_gVCF_strelka/aggV2",
                              "additional_data/PCs_relatedness/aggV2_PCs_ancestry_relatedness.tsv"))
    pcs %>%
      filter(ancestry0_8 == "EUR", unrelated_set == 1) %>%
      transmute(`#FID` = platekey, IID = platekey) %>%
      write_tsv("~/eur_unrelated.txt")
    ```

    Then test them, in the same Terminal (where `G` is still set):

    ```bash
    plink2 --bfile $G/genotypes_plink/aggV2_common_variants_extract --keep ~/eur_unrelated.txt --hardy --out ~/hwe_eur
    ```

3. Back in R, read both results with `read_tsv()`. In each, how many SNPs fail at p < 10<sup>−6</sup>, a threshold often used before a GWAS? Then compare the observed and expected proportions of heterozygotes, `` `O(HET_A1)` `` and `` `E(HET_A1)` `` (names like these need backticks in R): where they differ, in which direction? If the two sets of genomes give different answers, why?
4. *If you have time:* keeping one ancestry group and leaving out relatives are two changes at once. Make a list for each on its own, and find out which matters more.
5. *If you have time:* choose one SNP and test it by hand from its genotype counts (`HOM_A1_CT`, `HET_A1_CT`, `TWO_AX_CT`), with the chi-square test from the practical. Is your p-value close to PLINK’s?

## Task 6: your own project

*Choose the test for your primary question, and run it as you planned.*

1. **Choose the test**, with the table in the practical’s *Choosing a test*: what kind of outcome it is, how many groups you are comparing, and whether they are independent or paired.
2. **Write it down first.** Before you run it, make sure your plan says which test you will use, which estimate you will report, and what you will do if the test’s assumptions fail (for example, “if the groups are skewed, I will use the Wilcoxon test”). If that changes your registered plan, add a dated line to `changes.md`.
3. **Run it**, and report the estimate, its confidence interval, the test and the p-value in a sentence.
4. If your plan adjusts for other variables, this test is the unadjusted comparison: say so in your report. You will add the adjustment with regression in week 4.
5. Don’t try several tests to find the smallest p-value. If you run a second test as a check, report both, and say which one you planned.

## Your report

Put the tests that matter from each task into `week3.qmd`, each with a sentence that reports it: the estimate, its 95% confidence interval, the test and the p-value, for example “The mean difference was … (95% CI … to …; Welch’s t-test, p = …)”. Use inline code (`` `r …` ``) for the numbers, so that they always match the data. The simulator’s reports can’t show mathematical notation, so a very small number written with inline code appears as `6.3\times 10^{-8}`: write p-values with `format.pval(p, digits = 2)`, which gives `6.3e-08`.

Then **Render** it. Rendering fetches the tables again and repeats your shuffles, so it takes a minute or two, and it reads PLINK’s results from your home folder, so run PLINK first. If RStudio asks to install knitr and markdown, choose **Install**.

## Taking a result out (optional)

For practice, export a table of your results through the Airlock: for each test, the estimate, its confidence interval and the p-value, saved with `write_csv()`. Follow [the rules](../docs/airlock.md#what-may-leave): no counts below 5 (the sizes of the groups count too), and export it under the training project **RR4001**.

**Think about:** could any of your results, or the sizes of the groups behind them, give away a count below 5?
