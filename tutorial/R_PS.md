# Update R package

```{r}
library(data.table)

x <- old.packages()

pkg_update <- if (is.null(x)) {
  data.table(
    package = character(),
    current_version = character(),
    available_version = character()
  )
} else {
  as.data.table(x)[, .(
    package = Package,
    current_version = Installed,
    available_version = ReposVer
  )]
}

pkg_update
```

# PS

## Create ATE weight using SuperLearner

```{r}
SuperLearner::listWrappers()

SL_library = c(
  #"SL.bartMachine",
  "SL.biglasso",
  "SL.mean",
  "SL.glm",
  "SL.glmnet",
  #"SL.gam", # Always reporting error and long time and contribute to nothing
  #"SL.ranger", # Always reporting error and long time and contribute to nothing
  "SL.xgboost"
)

set.seed(20260805)

# Standard Super Learner chooses the ensemble based on predictive loss. Good treatment prediction does not guarantee good covariate balance.

# WeightIt provides Balance Super Learner, which selects the combination of candidate predictions to optimize balance directly.

# For example, minimize the largest absolute standardized mean difference:
  
start_time = Sys.time()
W_ate <- weightit(
  formula = copd_main ~ age_recruite + sex + income0 + deprive_index + 
                          ethnicity + smoking_status + diabetes + BMI + 
                          PA_MET_minutes_sum + fasting_time_0 + lipid_drug_use,
  data = d1,
  method = "super",
  estimand = "ATE",
  SL.library = SL_library,
  SL.method = "method.balance", # estimates the Super Learner combination using binomial log-likelihood
  criterion = "smd.max",

  cvControl = list(
    V = 5,
    stratifyCV = TRUE,
    shuffle = TRUE
  ),

  missing = "ind",
  stabilize = TRUE
)

Sys.time() - start_time # 4 min


d1[, ps_ate := W_ate$ps]
d1[, wt_ate := W_ate$weights]

# Standardize weights, you need to make sure that the average wt = 1 for each group,
# Otherwise, the weighted Table 1 would be wrong
d1[, wt_ate := wt_ate/sum(wt_ate)*.N,copd_main]
d1[,.(mean_ps = mean(wt_ate)),copd_main]


d1[]

# Check distribution of PS weights
d1[,sum(is.na(wt_ate))]
d1[,quantile(wt_ate, seq(0.95, 1, 0.001))]



options(scipen = 99)
W_ate$info$coef
W_ate$info$cvRisk

# Create weighted design
design_ate = svydesign(
  ids = ~1,
  weights = ~wt_ate,
  data = d1)
```

## Cox regression

```
# Preferred survey implementation
fit_cox <- svycoxph(
  Surv(time, event) ~ treatment,
  design = design_ate
)

summary(fit_cox)

beta_cox <- coef(fit_cox)["treatment"]
se_cox <- sqrt(vcov(fit_cox)["treatment", "treatment"])

cox_result <- data.frame(
  HR = exp(beta_cox),
  lower = exp(beta_cox - 1.96 * se_cox),
  upper = exp(beta_cox + 1.96 * se_cox),
  p_value = 2 * pnorm(
    abs(beta_cox / se_cox),
    lower.tail = FALSE
  )
)

cox_result

# Equivalent coxph() implementation
fit_cox_alt <- coxph(
  Surv(time, event) ~ treatment,
  data = d1,
  weights = sw_ate,
  robust = TRUE,
  x = TRUE
)

summary(fit_cox_alt)

ph_test <- cox.zph(fit_cox_alt)

ph_test
plot(ph_test)
```

## KM curves

```
km_weighted <- svykm(
  Surv(time, event) ~ treatment_f,
  design = design_ate,
  se = TRUE # standard-error calculation can be computationally and memory intensive for large datasets with many events
)

plot(
  km_weighted,
  pars = list(
    lty = c(1, 2),
    lwd = c(2, 2)
  ),
  xlab = "Follow-up time",
  ylab = "Survival probability"
)

legend(
  "bottomleft",
  legend = levels(dat$treatment_f),
  lty = c(1, 2),
  lwd = c(2, 2),
  bty = "n"
)

survey::svylogrank(
  Surv(time, event) ~ treatment_f,
  design = design_survival
)
```


## Regression modeling

```
### Weighted linear Regression
fit_linear <- svyglm(
  brain_idp ~ treatment,
  design = design_ate,
  family = gaussian())

### Further doubly robust model
fit_linear_adjusted <- svyglm(
  brain_idp ~ treatment + age + sex + smoking_status,
  design = design_ate,
  family = gaussian())

### Weighted Poisson regression
fit_poisson <- svyglm(
  event_count ~ treatment + offset(log(followup_py)),
  design = design_ate,
  family = quasipoisson(link = "log"))

### Weighted negative binomial regression
library(MASS)
library(sandwich)
library(lmtest)

fit_nb <- glm.nb(
  event_count ~ treatment + offset(log(followup_py)),
  data = count_dat,
  weights = design_ate,
  link = log)

vcov_nb <- sandwich::vcovHC(
  fit_nb,
  type = "HC0"
)

nb_test <- lmtest::coeftest(
  fit_nb,
  vcov. = vcov_nb
)

beta_nb <- coef(fit_nb)["treatment"]
se_nb <- sqrt(vcov_nb["treatment", "treatment"])

nb_result <- data.frame(
  IRR = exp(beta_nb),
  lower = exp(beta_nb - 1.96 * se_nb),
  upper = exp(beta_nb + 1.96 * se_nb),
  p_value = 2 * pnorm(
    abs(beta_nb / se_nb),
    lower.tail = FALSE
  )
)

nb_result

# If participants have multiple rows, Use participant-clustered sandwich variance
vcov_nb_cluster <- sandwich::vcovCL(
  fit_nb,
  cluster = count_dat$id,
  type = "HC0"
)

lmtest::coeftest(
  fit_nb,
  vcov. = vcov_nb_cluster
)
```



