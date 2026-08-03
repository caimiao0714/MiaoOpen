# DAG-based confounder selection

Assume that you are a world-class clinician, respiratory epidemiologist, neuroimaging researcher, and causal inference epidemiologist. We are conducting a population-based study using UK Biobank data to investigate the association between chronic obstructive pulmonary disease (COPD)/impaired lung function (defined using FEV1/FVC <0.70), compared to non-COPD/normal lung function (defined using FEV1/FVC >=0.70), and brain imaging-derived phenotypes (IDPs).

The exposure was dichotomous. The COPD group included participants with any evidence of COPD, defined as an inpatient diagnosis, self-reported COPD, a GP-recorded diagnosis, or airflow obstruction (FEV1/FVC <0.70). The non-COPD group included participants with no COPD diagnosis and preserved airflow, defined as FEV1/FVC ≥0.70.

The outcomes include multimodal brain imaging-derived phenotypes (IDPs), including: (1) structural MRI measures of global and regional brain volumes, cortical thickness, cortical surface area, and brain morphology; (2) diffusion MRI measures of white matter microstructural integrity, including fractional anisotropy, mean diffusivity, and tract-specific metrics; (3) cerebrovascular imaging markers, including white matter hyperintensity burden and other vascular-related brain abnormalities; (4) resting-state functional MRI measures, including functional connectivity, network organization, and intrinsic brain activity.

Using a causal inference framework and directed acyclic graph (DAG) methodology, please identify and classify all potential confounders that may influence the relationship between COPD and IDPs and we should adjust for them in our regression models.

For each candidate covariate, please: Evaluate whether it is a true confounder based on causal relationships rather than statistical association. Determine whether and how the variable influences both exposure and outcome. Assess whether the variable should be adjusted for in the primary analysis, included only in sensitivity analyses, or not adjusted for due to risk of over-adjustment or collider bias. Assign each variable to an appropriate biological or methodological domain.


