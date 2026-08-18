# Create R scripts and `_batch_list.txt`

Below is R code to generate:
1. xxx R files for each outcome using a template `_template.R`
2. `_batch_list.txt` contains a list of all R scripts to run
3. `log` folder to record logs

```{r}
pacman::p_load(glue, stringr, readr, fst, data.table)

file_path = '/mnt/data1/Users/hy001'

# 1. Create outcome names
d_name = read_fst('/mnt/data1/Users/hy001/metabolite_names.fst', as.data.table = TRUE)
nmr_vars <- d_name$metabolite_name

# 2. Read template R script
r_template = readr::read_file(glue('{file_path}/Batch/2026-08-18_batch_lm/Code/_template.R'))

# 3. Create R script for each outcome
for (i in 1:length(nmr_vars)) {
  id_pad = stringi::stri_pad_left(i, width = 3, pad = 0)
  
  cat(glue("=========== {id_pad}/{length(nmr_vars)} ==========="))
  
  r_gen = glue("outcome = '{nmr_vars[i]}'\n{r_template}") # add outcome specification
  
  write_file(r_gen, glue("{file_path}/Batch/2026-08-18_batch_lm/Code/{id_pad}_{nmr_vars[i]}.R"))
}

# 4. Generate _batch_list.txt
data.table(file_name = list.files(glue('{file_path}/Batch/2026-08-18_batch_lm/Code/'), 
                       pattern = '\\d+.*\\.R$')) %>% 
  write_delim(glue("{file_path}/Batch/2026-08-18_batch_lm/Code/_batch_list.txt"),
              col_names = FALSE,
              #delim = "\t",
              quote = 'none')

# 5. Create a log folder
fs::dir_create(glue("{file_path}/Batch/2026-08-18_batch_lm/Code/log"))
```

### manifest

```
#!/bin/bash

#SBATCH --job-name=R_batch
#SBATCH --cpus-per-task=1
#SBATCH --mem=8G
#SBATCH --time=12:00:00
#SBATCH --output=logs/%A_%a.out
#SBATCH --error=logs/%A_%a.err

# Go to the directory from which sbatch was submitted
cd "$SLURM_SUBMIT_DIR"

# Manifest file containing one R script per line
MANIFEST="_batch_list.txt"

# Read the R script corresponding to this array task
SCRIPT=$(sed -n "${SLURM_ARRAY_TASK_ID}p" "$MANIFEST" | tr -d '\r')

# Print basic job information
echo "=============================================="
echo "SLURM job ID:       $SLURM_JOB_ID"
echo "SLURM array job ID: $SLURM_ARRAY_JOB_ID"
echo "SLURM task ID:      $SLURM_ARRAY_TASK_ID"
echo "R script:           $SCRIPT"
echo "Host:               $(hostname)"
echo "Working directory:  $(pwd)"
echo "Start time:         $(date)"
echo "=============================================="

# Check that the manifest returned a script name
if [[ -z "$SCRIPT" ]]; then
    echo "ERROR: No script found for task $SLURM_ARRAY_TASK_ID"
    exit 1
fi

# Check that the R script exists
if [[ ! -f "$SCRIPT" ]]; then
    echo "ERROR: R script does not exist: $SCRIPT"
    exit 1
fi

# Run the R script
Rscript "$SCRIPT"

STATUS=$?

echo "=============================================="
echo "R script:   $SCRIPT"
echo "Exit code:  $STATUS"
echo "End time:   $(date)"
echo "=============================================="

exit $STATUS
```



# Test run

```
cd /mnt/data1/Users/hy001/Batch/2026-08-18_batch_lm/Code
sbatch --array=1-3 _manifest.slurm
```

Confirm
- logs 正常
- R packages 正常
- 输出文件正常

# Submit and run batch


```
cd /mnt/data1/Users/hy001/Batch/2026-08-18_batch_lm/Code

N=$(awk 'END {print NR}' _batch_list.txt)

echo "$N"

sbatch --array=1-${N}%100 _manifest.slurm

```

# Check running/pending batch

```
squeue -u $USER
```

# Check completed batch

```
sacct -j 9 \
  --format=JobID,JobName,State,Elapsed,MaxRSS,ExitCode
```

(base) root@miaolab:/mnt/data1/Users/hy001/Batch/2026-08-18_batch_lm/Code# sacct -j 251 \
  --format=JobID,JobName,State,Elapsed,MaxRSS,ExitCode
Slurm accounting storage is disabled

# Combine output csv files

```
result = list.files('/mnt/data1/Users/hy001/Batch/2026-08-18_batch_lm/Results/', 
                    pattern = '^lm_.*\\.csv$', 
                    full.names = TRUE) %>% 
  vroom::vroom() %>% 
  setDT()

result
```
