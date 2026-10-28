# Week 4: regression

[All tasks](README.md) · [Back to the guide](../README.md)

In this week’s [statistics practical](https://comparativechrono.github.io/statistics-training/week4.html) you met correlation and linear regression, used genotypes as predictors, adjusted for confounders, looked for an interaction, checked a model with its residuals, and fitted a logistic regression. These tasks fit the same models to the Research Environment’s data, including genotypes from the genotype files. Work in RStudio and the Terminal in the simulator, and keep the models that matter in a Quarto report.

Several tasks touch questions from the [week 1 list](week-01-pre-registration.md#questions-you-could-ask). If one of them is your own project’s question, do it the way your plan says rather than the way the task says, and note anything you do differently in `changes.md`.

**Time:** about three hours. Parts marked *If you have time* are extra.\
**You will use:** RStudio, and the Terminal for PLINK 2 in tasks 2 and 5.\
**You will hand in:** nothing this week, unless your course team asks. There is an optional Airlock export at the end, for practice.

## Before you start

1. Open the simulator, **RStudio** and a **Terminal**. Your `~/.netrc` and the tidyverse from earlier weeks are still there (if not, see [week 2](week-02-describing-data.md#before-you-start)).
2. Start your report: **File ▸ New File ▸ Quarto Document…**, saved as `week4.qmd` in your project folder, `/re_gecip/training/<your username>`, with the same setup chunk as last week (copy it from your `week3.qmd`). Run it.
3. In the Terminal, load PLINK 2 and name the genotype files:

    ```bash
    module load plink2/2.0.0-b.1-dev
    G=/gel_data_resources/main_programme/aggregation/aggregate_gVCF_strelka/aggV2/additional_data/genotypes_plink/aggV2_common_variants_extract
    ```

**Two habits for models.** Say what each predictor counts: which allele a genotype counts, which group is the reference, and the units of a number. And report each coefficient (or odds ratio) with its 95% confidence interval and p-value, and say what the model adjusts for. The habits of weeks 2 and 3 still apply: what one row represents, coded values, and looking before you test.

## Task 1: correlation and a regression line

*Do older fathers pass on more de novo mutations?*

- **Tables:** `denovo_cohort_information`, `participant`
- **Methods:** scatter plots, Pearson’s and Spearman’s correlation, `lm()`, `confint()`, `predict()`

1. **The data.** As in week 3, work out each parent’s age at the child’s birth, and keep the trio’s de novo counts:

    ```r
    trios <- lk("denovo_cohort_information", c("trio_id", "family_id", "participant_id", "member",
                                               "base_filter_total", "stringent_filter_total"))
    born <- lk("participant", c("participant_id", "year_of_birth"))
    dnm <- trios %>%
      left_join(born, by = "participant_id") %>%
      select(-participant_id) %>%
      pivot_wider(names_from = member, values_from = year_of_birth) %>%
      mutate(father_age = Offspring - Father, mother_age = Offspring - Mother,
             dn = your_count)
    ```

    Replace `your_count` with the count you chose in week 2.
2. **Look.** Draw a scatter plot of `dn` against `father_age` (`geom_point(alpha = 0.3)`). Is the relationship roughly a straight line?
3. **Correlation.** `cor.test(dnm$father_age, dnm$dn)`, and Spearman’s correlation with `method = "spearman"`. What proportion of the variation in `dn` goes with the father’s age?
4. **A regression line.** Fit `fit <- lm(dn ~ father_age, data = dnm)`, and look at `summary(fit)` and `confint(fit)`. By how much does a child’s average number of de novo mutations change for each year of the father’s age? What does the intercept mean? Predict the average for fathers aged 25 and 45 with `predict(fit, newdata = tibble(father_age = c(25, 45)))`, and draw the line with `geom_smooth(method = "lm")`.

**Think about:** correlation isn’t causation. Is there another explanation for what you found? (Task 3 tests one.)

## Task 2: genotypes as predictors

*Body mass index and FTO.*

- **Tables and files:** `cancer_risk_factor_general`, `participant`, the genotype files and the ancestry file
- **Methods:** the additive, genotypic, dominant and recessive codings; `lm()` with covariates

1. **The genotypes.** chr16:53786615:T:A is rs9939609, a SNP in *FTO* (the IDs are chromosome:position:REF:ALT). In the Terminal, write its genotypes as numbers, 0, 1 or 2:

    ```bash
    plink2 --bfile $G --snp chr16:53786615:T:A --export A --out ~/fto
    ```

2. **Link them to participants.** `~/fto.raw` has a row for each genome, named by its plate key; the ancestry file links plate keys to participant IDs. Keep the unrelated genomes, because relatives aren’t independent:

    ```r
    fto <- read_tsv("~/fto.raw")
    pcs <- read_tsv(file.path("/gel_data_resources/main_programme/aggregation/aggregate_gVCF_strelka/aggV2",
                              "additional_data/PCs_relatedness/aggV2_PCs_ancestry_relatedness.tsv"),
                    col_types = cols(participant_id = "c"))
    geno <- fto %>%
      select(platekey = IID, g = last_col()) %>%
      inner_join(filter(pcs, unrelated_set == 1), by = "platekey") %>%
      select(participant_id, g)
    ```

    Which allele does `g` count? Find out, and say so whenever you report its effect.
3. **BMI.** Make `bmi`, a table with `participant_id` and `bmi`, from `cancer_risk_factor_general` as you did in week 2, with the same cleaning decisions. Then add the genotypes and each person’s sex and age:

    ```r
    people <- lk("participant", c("participant_id", "participant_phenotypic_sex", "year_of_birth")) %>%
      mutate(age = 2024 - year_of_birth)
    bmi_fto <- bmi %>%
      inner_join(geno, by = "participant_id") %>%
      left_join(people, by = "participant_id")
    ```

4. **The additive model.** Look at the mean BMI for each genotype (`group_by(g)` and `summarise()`), then fit `lm(bmi ~ g + participant_phenotypic_sex + age, data = bmi_fto)`. How much does BMI change for each copy of the allele that `g` counts? How much of the variation in BMI does the genotype explain?
5. **Other codings.** Fit the genotypic model (`factor(g)`), and the dominant and recessive models (`I(g >= 1)` and `I(g == 2)`, as in the practical). Which coding describes the pattern of means best?

## Task 3: confounding and interactions

*Allow for other variables, and ask whether an effect differs between groups.*

- **Tables:** your `dnm` and `bmi_fto`, and `cancer_analysis`
- **Methods:** multiple regression, confounding, interactions

1. **Both parents.** A child’s de novo mutations could come from either parent. Fit `dn` on `mother_age` alone, then on `father_age + mother_age`. What happens to each coefficient when both are in the model? Does the correlation between the parents’ ages help to explain any change?
2. **Sample preparation, again.** In week 3 you compared tumour mutational burden between fresh-frozen and FFPE samples, and asked what else could differ between them. Fit the comparison as a model, and then allow for cancer type:

    ```r
    cancer <- lk("cancer_analysis", c("participant_id", "disease_type", "study_abbreviation",
                                      "preparation_method", "somatic_coding_variants_per_mb")) %>%
      mutate(log_tmb = log10(somatic_coding_variants_per_mb))
    prep <- cancer %>% filter(preparation_method %in% c("FF", "FFPE"))
    fit1 <- lm(log_tmb ~ preparation_method, data = prep)
    fit2 <- lm(log_tmb ~ preparation_method + study_abbreviation, data = prep)
    ```

    Compare the coefficient `preparation_methodFFPE` in the two models. On the log10 scale, `10^` of a coefficient and of its confidence interval is a ratio. Is cancer type a confounder here, and how can you tell?
3. **An interaction.** Does the effect of the FTO genotype differ between women and men? Fit `lm(bmi ~ g * participant_phenotypic_sex + age, data = bmi_fto)`, and read the interaction term as in the practical.

**Think about:** a test for an interaction has much less power than the test of a main effect. If you found no interaction, what can you conclude? And if you had looked for interactions with ten other variables, how would you read one with p = 0.038?

## Task 4: checking a model

*Look at the residuals before you trust the p-values.*

- **Tables:** your `prep` and `dnm`
- **Methods:** residuals against fitted values, QQ plots of residuals, LINE

1. **On the raw scale.** Fit the model of task 3, step 2, to the raw tumour mutational burden: `fit_raw <- lm(somatic_coding_variants_per_mb ~ preparation_method + study_abbreviation, data = prep)`. Plot its residuals:

    ```r
    diag <- tibble(fitted = fitted(fit_raw), residual = residuals(fit_raw))
    ggplot(diag, aes(x = fitted, y = residual)) +
      geom_point(alpha = 0.3) +
      geom_hline(yintercept = 0)
    ggplot(diag, aes(sample = residual)) +
      stat_qq() +
      stat_qq_line()
    ```

2. **On the log scale.** Draw the same plots for `fit2`, the model of `log_tmb`. Which model’s assumptions are more reasonable, and which would you report?
3. **De novo mutations.** Check the residuals of `lm(dn ~ father_age + mother_age, data = dnm)` the same way. Then think about the I of LINE: can a family appear more than once in `dnm` (week 3, task 2)? If it can, are the trios independent?
4. *If you have time:* tests are models. Compare `t.test(log_tmb ~ preparation_method, data = prep, var.equal = TRUE)` with `summary(fit1)`, and the ANOVA of week 3 with `anova(lm(log_tmb ~ study_abbreviation, data = cancer))`.

## Task 5: logistic regression

*Smoking and lung cancer.*

- **Tables:** `cancer_analysis` (your `cancer`), `cancer_risk_factor_general`, `participant` (your `people`); for the extra, the genotype files
- **Methods:** `glm(family = binomial)`, log odds and odds ratios, `confint()`

1. **The data.** Among the cancer participants, the outcome is whether the cancer is a lung cancer:

    ```r
    smoking <- lk("cancer_risk_factor_general", c("participant_id", "smoking_status"))
    smokers <- cancer %>%
      left_join(smoking, by = "participant_id") %>%
      left_join(people, by = "participant_id") %>%
      mutate(lung = disease_type == "LUNG")
    ```

    Add a column `ever`: TRUE for current and ex-smokers, FALSE for people who never smoked. Decide, as in week 2, what to do with any other values.
2. **The model.** Fit `m <- glm(lung ~ ever + participant_phenotypic_sex + age, family = binomial, data = smokers)`, look at `summary(m)`, and turn its coefficients into odds ratios with confidence intervals: `exp(cbind(OR = coef(m), confint(m)))`. Write the odds ratio for smoking as a sentence.
3. *If you have time:* chr15:78590583:G:A is rs16969968, in *CHRNA5*. Export its genotypes as you did for FTO, add them to the model, and then ask whether their effect differs between people who have and haven’t smoked (`chrna5 * ever`). Report the interaction with its confidence interval, and say what that interval does and doesn’t rule out.

**Think about:** the people without lung cancer here are people with other cancers, not the general population. How might that affect the odds ratio for smoking?

## Task 6: your own project

*Fit the model in your plan.*

1. **Fit it** as your plan says: the outcome, the exposure, and the confounders you named, with `lm()` for a continuous outcome or `glm(family = binomial)` for a yes-or-no one. If the plan has changed, add a dated line to `changes.md`.
2. **Check it.** For a linear model, plot the residuals. For a logistic model, count the events: as a rough guide, you need 10 to 20 for each predictor.
3. **Report it:** the coefficient or odds ratio of your exposure, its confidence interval and p-value, what it is adjusted for, and what the exposure counts.
4. **Say what adjusting changed.** Compare the adjusted estimate with the unadjusted one from week 3. Don’t adjust for variables that your exposure itself affects.

## Your report

Put the models that matter from each task into `week4.qmd`, each with a sentence that reports it, for example “Each copy of the … allele was associated with a difference in BMI of … kg/m² (95% CI … to …; p = …), adjusted for sex and age.” Use inline code for the numbers, and `format.pval()` for p-values (see [week 3](week-03-testing-hypotheses.md#your-report)). The report reads `~/fto.raw`, so run PLINK first. Then **Render** it.

## Taking a result out (optional)

For practice, export a table of your models’ estimates through the Airlock: for each, the coefficient or odds ratio, its confidence interval and the p-value, saved with `write_csv()`. Follow [the rules](../docs/airlock.md#what-may-leave), and export it under the training project **RR4001**.

**Think about:** which of your results could leave as they are, and which would need the sizes of their groups checked first?
