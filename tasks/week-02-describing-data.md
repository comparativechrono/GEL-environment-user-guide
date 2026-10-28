# Week 2: describing data, distributions and uncertainty

[All tasks](README.md) · [Back to the guide](../README.md)

In this week’s [statistics practical](https://comparativechrono.github.io/statistics-training/week2.html) you described the FAMuSS data, looked at distributions, and put standard errors and confidence intervals on your estimates. These tasks use the same skills on the Research Environment’s data, where nobody has tidied the tables for you. Work in RStudio in the simulator, and keep the code that matters in a Quarto report, as you did in the practical.

**Time:** about two to three hours. Parts marked *If you have time* are extra.\
**You will use:** LabKeeper’s Schema Browser, RStudio, and the Terminal for one extra task.\
**You will hand in:** nothing this week, unless your course team asks. There is an optional Airlock export at the end, for practice.

## Before you start

1. Open the simulator, sign in as usual, and open **RStudio**. If you haven’t set up `~/.netrc` yet, do it now: see [The data](../docs/data.md#from-r).
2. Load the tidyverse with `library(tidyverse)`. If R says there is no package called ‘tidyverse’, install it first with `install.packages("tidyverse")`; it takes about half a minute.
3. Start your report: **File ▸ New File ▸ Quarto Document…**, and save it as `week2.qmd` in your project folder, `/re_gecip/training/<your username>`. Give it this setup chunk, which loads the packages and defines `lk()`, a short way to fetch a table from LabKeeper:

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

    Run it with **Run Current Chunk** (the ▶ at the chunk’s top right, or Ctrl+Shift+Enter), so that the same objects are in your Console.

You can’t paste code in from this page, so type it. Inside the RE, copy and paste work: LabKeeper’s **Export ▸ Script** writes the R code that fetches the table you are looking at, and you can copy it into RStudio.

**Two habits for these data.** Before you count anything, find out what one row of a table represents: one participant, one sample, one tumour, one variant, one hospital episode? And read each column’s label and coded values in LabKeeper’s **Schema Browser** before you use it.

## Task 1: what kind of data are they?

*Describe some of the main variables of the cohort, as you described FAMuSS in the practical.*

- **Tables:** `participant`, `cancer_analysis`, `cancer_staging_consolidated`, `cancer_risk_factor_general`, `rare_diseases_participant_phenotype`
- **Methods:** kinds of data; centre and spread; missing values; summaries by group

```r
participant <- lk("participant", c("participant_id", "programme", "participant_type",
                                   "participant_phenotypic_sex", "year_of_birth"))
cancer <- lk("cancer_analysis", c("participant_id", "study_abbreviation",
                                  "somatic_coding_variants_per_mb"))
stage <- lk("cancer_staging_consolidated", c("participant_id", "stage_best"))
risk <- lk("cancer_risk_factor_general", c("participant_id", "smoking_status", "height", "weight"))
```

1. **What kind of data?** Decide what kind each of these is (binary, categorical, ordinal, a count or continuous): sex, year of birth, cancer type (`study_abbreviation`), tumour mutational burden (`somatic_coding_variants_per_mb`, the number of somatic coding mutations per megabase), stage (`stage_best`), smoking status, height and weight. Summarise each one in the way that suits it: `count()` for the categories, `summary()` for the numbers.
2. **Missing and impossible values.** Count the missing values in each (`sum(is.na(x))`). Then look for values that stand for “missing” without being `NA`, and for values that can’t be right: look at the smallest and largest values, and at the coded values in the Schema Browser. Decide what you will do with each, and write your decisions in your report.
3. **A count.** Make the number of HPO terms recorded as present for each rare-disease proband, and summarise it:

    ```r
    probands <- participant %>%
      filter(programme == "Rare Diseases", participant_type == "Proband")
    hpo <- lk("rare_diseases_participant_phenotype", c("participant_id", "hpo_present"))
    n_terms <- hpo %>%
      filter(hpo_present == "Yes") %>%
      count(participant_id, name = "terms") %>%
      right_join(select(probands, participant_id), by = "participant_id") %>%
      mutate(terms = replace_na(terms, 0))
    ```

    The `right_join()` and `replace_na()` keep the probands who have no terms recorded as present, with a count of 0: without them, those probands would disappear.
4. **Summaries by group.** For each cancer type: the number of tumours, and the median and IQR of tumour mutational burden (`group_by()` and `summarise()`, as in the practical).
5. **Body mass index** is weight in kilograms divided by the square of height in metres. Calculate it once you have dealt with the values you found in step 2, then give the mean, SD, median and IQR of BMI for women and for men (join `risk` to `participant` by `participant_id` to get their sex).

**Think about:**

- Which would you report for tumour mutational burden: the mean and SD, or the median and IQR? Why?
- When a variable has a category that means “not known”, is that a category, or a missing value? What difference does your choice make?
- How many people did you leave out of the BMI summary, and why? Say so in your report.

## Task 2: seeing distributions

*Plot the distributions before you summarise or test anything.*

- **Tables:** `cancer_analysis`, `denovo_cohort_information`, and your BMI from task 1
- **Methods:** histograms, box plots, density plots, log scales

1. Draw a histogram of tumour mutational burden. Then draw it again on a log scale, by adding `+ scale_x_log10()`. What does the log scale show that the first plot hides?
2. Compare tumour mutational burden between the commoner cancer types with box plots, on a log scale:

    ```r
    common <- cancer %>% count(study_abbreviation) %>% filter(n >= 100) %>% pull(study_abbreviation)
    cancer %>%
      filter(study_abbreviation %in% common) %>%
      ggplot(aes(x = study_abbreviation, y = somatic_coding_variants_per_mb)) +
      geom_boxplot() +
      scale_y_log10()
    ```

3. **De novo mutations per child.** `denovo_cohort_information` has one row for each member of a trio (the child, the mother and the father), with the trio’s counts repeated on each row, so keep one row per trio:

    ```r
    trios <- lk("denovo_cohort_information") %>% filter(member == "Offspring")
    ```

    It has two counts of de novo mutations, from a base filter and a stringent filter (`base_filter_total`, `stringent_filter_total`). Plot the distribution of each, describe their shapes, and decide which you would use.
4. Draw density plots of BMI for women and for men, on the same axes (`geom_density(alpha = 0.5)`, as in the practical).

**Think about:** Which distributions are roughly symmetric, and which are skewed? Does anything look odd: more than one peak, a spike, a long tail? What could cause it?

## Task 3: probability distributions

*Compare the data with the distributions from the practical.*

- **Tables:** `denovo_cohort_information` (your `trios` from task 2), `cancer_analysis`
- **Methods:** the Poisson, binomial and normal distributions; simulating data

1. **Poisson.** De novo mutations arise independently of each other, so their number in each child might follow a Poisson distribution. Using the count you chose in task 2, compare its mean with its variance. Then draw the Poisson distribution with the same mean over the histogram (replace `your_count` with the column you chose):

    ```r
    dn <- pull(trios, your_count)
    pois <- tibble(x = min(dn):max(dn), p = dpois(min(dn):max(dn), lambda = mean(dn)))
    ggplot(tibble(dn), aes(x = dn)) +
      geom_histogram(aes(y = after_stat(density)), binwidth = 1) +
      geom_line(data = pois, aes(x = x, y = p), colour = "red")
    ```

    Is the variance close to the mean? If it is much larger, the counts are over-dispersed: what might make some children have more de novo mutations than others?
2. **Binomial.** Suppose a genome test finds a diagnosis for 1 in 5 probands, and a clinic tests 60 probands a year. How many diagnoses would you expect in a year? Use `dbinom()` for the probabilities, and simulate 1,000 years with `rbinom()`. What range of yearly numbers would not surprise you?
3. **Normal.** Choose one cancer type. Is the log of its tumour mutational burden close to a normal distribution? Plot its histogram with a normal curve of the same mean and SD (`dnorm()`), or compare it with data simulated with `rnorm()`.
4. *If you have time:* simulate as many children as there are trios, with Poisson counts at the mean you found, and compare the histogram of your simulated counts with the real one.

## Task 4: how sure can we be?

*Put confidence intervals on your estimates.*

- **Tables:** `gmc_exit_questionnaire`, `rare_diseases_participant_disease`, `cancer_analysis`, your BMI from task 1
- **Methods:** standard errors, confidence intervals for a mean and a proportion, the bootstrap

1. **Diagnostic yield.** What proportion of rare-disease probands had their case solved? The clinical teams’ verdicts are in the `case_solved_family` column of `gmc_exit_questionnaire`. Before you calculate it, decide how you will count: what does one row of the table represent, and which of the column’s answers will you count as solved, which as not solved, and which, if any, will you leave out? Then give the yield with its 95% confidence interval, using the formula from the practical, p ± 1.96 √(p(1 − p)/n), or `binom.test(x, n)$conf.int`.
2. **Yield by disease group.** Join the probands to `rare_diseases_participant_disease` and calculate the yield, with its confidence interval, for each `normalised_disease_group`. Plot the yields with their intervals (`geom_pointrange()`). Which groups have the widest intervals, and why?
3. **A mean.** Give the mean BMI with its 95% confidence interval (`t.test(bmi)$conf.int`), for women and for men.
4. **The bootstrap.** Choose one cancer type and give the median tumour mutational burden with a 95% bootstrap confidence interval, as in the practical: `replicate()` the median of `sample(x, replace = TRUE)` 2,000 times, and take the 2.5% and 97.5% `quantile()`s. Why use the bootstrap here, rather than a formula?
5. *If you have time:* **an allele’s frequency.** In the Terminal, count the alleles of every SNP in the genotype files:

    ```bash
    module load plink2/2.0.0-b.1-dev
    G=/gel_data_resources/main_programme/aggregation/aggregate_gVCF_strelka/aggV2/additional_data
    plink2 --bfile $G/genotypes_plink/aggV2_common_variants_extract --freq counts --out ~/all
    ```

    Read `~/all.acount` into R with `read_tsv()`. For a SNP of your choice, the frequency of its ALT allele is `ALT_CTS / OBS_CT`, and its 95% confidence interval is p ± 1.96 √(p(1 − p)/`OBS_CT`), because `OBS_CT` counts alleles, two per person. Then compare two ancestry groups. Make a list of the genomes in each group from the ancestry file, and give it to PLINK with `--keep`; for example, for the genomes of European ancestry:

    ```r
    pcs <- read_tsv(file.path("/gel_data_resources/main_programme/aggregation/aggregate_gVCF_strelka/aggV2",
                              "additional_data/PCs_relatedness/aggV2_PCs_ancestry_relatedness.tsv"))
    pcs %>%
      filter(ancestry0_8 == "EUR") %>%
      transmute(`#FID` = platekey, IID = platekey) %>%
      write_tsv("~/eur.txt")
    ```

    ```bash
    plink2 --bfile $G/genotypes_plink/aggV2_common_variants_extract --keep ~/eur.txt --freq counts --out ~/eur
    ```

    Do the same for a second group, and compare the allele’s frequency in the two. Do their confidence intervals overlap?

## Task 5: your own project

*Describe the variables of the project you registered in week 1.*

1. Fetch the variables your plan uses, and describe each one **on its own**: what kind of data it is, a suitable summary, its missing and impossible values, and a plot of its distribution. Don’t look at your outcome by your exposure: that comes after your plan is settled.
2. Count the people who have every variable your analysis needs. That is the number your sample size and power should be based on.
3. Does anything you found change your plan? For example, a skewed outcome may call for a log scale or the median; many missing values need a plan; a category with fewer than 5 people will have to be merged before you can report it. If you change the plan, add a dated line to `changes.md` saying what changed and why.
4. Add a section to your report that describes your project’s data.

## Your report

Put the code that matters from each task into `week2.qmd`, with a sentence or two about what it shows, and **Render** it. Render runs the whole document in a new R, so the report must fetch its own data, which the setup chunk does; fetching the tables again takes a minute or two. The first time, RStudio asks to install the packages knitr and markdown, which do the rendering: choose **Install**. Use inline code (`` `r …` ``) for the numbers in your text, so that they always match the data. Keep the report in your project folder: it is the record of your work.

## Taking a result out (optional)

For practice with real results, export one table through the Airlock: for example, your yields by disease group, with their confidence intervals, saved with `write_csv()`. Follow [the rules](../docs/airlock.md#what-may-leave): mask counts below 5 as `<5`, check that no total gives a masked count away, and export it under the training project **RR4001**. The rendered report is an HTML file, which the Airlock asks to be converted to PDF, so export the table rather than the report.

**Think about:** if you mask a count, can it still be worked out from the proportion or the confidence interval next to it?
