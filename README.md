# qsubsh2

A simple job submitter for PBS Pro, Torque, SGE and Slurm.

`qsubsh2.sh` detects the cluster type automatically and submits jobs with the correct backend command without making you rewrite scheduler-specific headers such as `#PBS` or `#SBATCH`. It keeps the same calling style as classic [qsubshcom](https://github.com/zhilizheng/qsubshcom), but improves submission failure handling, submission logging, temporary script cleanup, and runtime resource monitoring.

It is designed for the common workflow of:

- submitting one command quickly
- chaining jobs with dependencies
- running job arrays with a cluster-independent `{TASK_ID}`
- keeping a submission history in the working directory
- collecting stdout, stderr, and lightweight resource monitoring under `./job_reports`

Author: Zhili-inspired workflow, adapted version for `qsubsh2.sh`  
License: MIT-style workflow reference, adjust to your repository license as needed.

## Installation

Clone this repository or copy `qsubsh2.sh` to a location in your `PATH`.

```bash
chmod +x qsubsh2.sh

# optional: put it somewhere in PATH
cp qsubsh2.sh ~/bin/qsubsh2

# test the command
qsubsh2 "echo hello" 1 1G demo 00:01:00 ""
```

If you prefer to run it locally from the repository:

```bash
./qsubsh2.sh "echo hello" 1 1G demo 00:01:00 ""
```

## Usage

```bash
# qsubsh2.sh "command1 |; command2" num_CPU total_memory task_name run_time other_params
# logs go to:
#   ./job_reports/*.log   stdout
#   ./job_reports/*.err   stderr
#   ./job_reports/*.mon   monitor information
# submission history goes to:
#   ./qsub.YYYY.MM.DD.log
```

The interface is intentionally unchanged from the older style:

```bash
qsubsh2.sh "echo 'hello world'" 1 1G helloTask 1:00:00 ""
```

## Common Calling Patterns

### Submit a single command

```bash
qsubsh2.sh "echo 'hello world'" 1 1G helloTask 1:00:00 ""
```

### Submit multiple commands in one job

Use `|;` to split commands into separate lines in the generated job script.

```bash
qsubsh2.sh "echo 'step1' |; echo 'step2' |; echo 'step3'" 1 1G multiStep 1:00:00 ""
```

### Run an existing shell script

```bash
qsubsh2.sh "bash test_script.sh" 1 1G demoSh 1:00:00 ""
```

You can also run it directly if it is executable:

```bash
qsubsh2.sh "./test_script.sh" 1 1G demoSh 1:00:00 ""
```

### Run Python or R

```bash
qsubsh2.sh "python3 test.py" 1 1G demoPy 1:00:00 ""

qsubsh2.sh "Rscript test.R --chr={TASK_ID}" 1 1G demoR 1:00:00 "-array=1-22"
```

### Run a job array

`{TASK_ID}` is replaced with the scheduler-specific array index variable at runtime.

```bash
qsubsh2.sh "echo 'hello world {TASK_ID}' > {TASK_ID}.log" 1 1G helloArray 1:00:00 "-array=1-10"
```

### Chain jobs with dependencies

```bash
job1=$(qsubsh2.sh "echo 'hello world 1'" 1 1G helloTask1 1:00:00 "")
job2=$(qsubsh2.sh "echo 'hello world 2'" 1 1G helloTask2 1:00:00 "")

job3=$(qsubsh2.sh "echo 'hello again'" 1 1G helloTask3 1:00:00 "-wait=$job1:$job2")

qsubsh2.sh "echo 'array task {TASK_ID}' > out_{TASK_ID}.txt" 1 1G helloTask4 1:00:00 "-wait=$job3 -array=1-10"
```

### Specify queue, account, or engine manually

```bash
qsubsh2.sh "python3 test.py" 4 8G myJob 12:00:00 "-queue=batch -acct=my_project"

qsubsh2.sh "python3 test.py" 4 8G myJob 12:00:00 "-ge=SLM"
```

### Pass scheduler-native extra parameters

Any extra parameters that are not reserved by `qsubsh2.sh` are passed through to `qsub` or `sbatch`.

```bash
qsubsh2.sh "hostname" 1 1G hostCheck 00:10:00 "-l host=host1:host2"
```

Note: these backend-specific parameters may not work across different cluster engines.

## Working With Existing Scripts

If you already have scripts, you can submit them as normal commands:

```bash
qsubsh2.sh "bash Your_script.sh" 2 4G your_task_name 10:00:00 ""
```

`qsubsh2.sh` submits a generated wrapper script from the current working directory, so an extra `cd WORK_DIR` is usually unnecessary.

If your original script contains scheduler-specific variables such as `${PBS_ARRAY_INDEX}` or `${SLURM_ARRAY_TASK_ID}`, replace them with `${TASK_ID}` when you want the same script to run across supported cluster types.

## Escaping Special Characters

If you want special characters such as `$` to be evaluated on the compute node instead of the submission shell, escape them in the command string.

A practical pattern is to store the command in a temporary shell variable first:

```bash
temp_command="awk '{print \$1, \$2}' test.txt > test2.txt"
qsubsh2.sh "$temp_command" 1 1G test_awk 00:00:05 ""
```

## Memory

The memory argument is the total memory requested for the task, not per CPU core.

For example:

```bash
qsubsh2.sh "echo hello" 3 5G test 00:00:05 ""
```

requests 3 CPU cores and 5 GB memory in total.

For Torque and SGE, the per-core memory request may be rounded up internally when the total memory is not divisible by the number of CPUs. That means a request like `3 cores + 5G total` can become effectively 6G total on those backends.

Use memory values like `1G`, `500M`, or `100K`. Avoid `GB` and `MB` because some schedulers do not accept them.

## Logs

`qsubsh2.sh` writes logs in two places:

- `./qsub.YYYY.MM.DD.log`: submission history, submit command, raw scheduler output, and parsed job id
- `./job_reports/`: runtime logs for each job

The files under `./job_reports/` include:

- `*.log`: stdout
- `*.err`: stderr
- `*.mon`: resource monitor output

The monitor records lightweight process information every 10 seconds, including:

- elapsed time
- CPU seconds
- `%CPU`
- RSS
- VSZ
- `VmHWM`

If submission fails, the failure is also written to the submission log and the script exits with a non-zero status.

## Other Parameters

The sixth argument, `other_params`, supports the following built-in options:

- `-wait=JOB1:JOB2:JOB3` wait for these jobs to finish successfully before starting
- `-array=1-100:2` create a job array such as `1, 3, 5, ..., 99`
- `-log=log_name` write submission history to `log_name.YYYY.MM.DD.log`
- `-ntype=node_type` specify node type for Torque-style clusters
- `-queue=queue_name` specify queue or partition
- `-acct=ACCOUNT_NAME` specify account, such as PBS `-A` or Slurm `-A`
- `-ge=CLUSTER_ENGINE` manually set the engine: `PBS`, `SGE`, `TOR`, or `SLM`

You can also append cluster-specific defaults through `QSUBSHCOM_EXTRAS`:

```bash
export QSUBSHCOM_EXTRAS="-acct=my_project -queue=batch"
```

These values are appended to the sixth argument automatically.

## End-to-End Example

```bash
chrs="1-22"
geno="test_chr{TASK_ID}"
pheno="pheno.txt"
snplist="chr{TASK_ID}.snplist"

genoQC1="extract_chr{TASK_ID}"
assocOut="test_assoc_chr{TASK_ID}"

cmd1="plink2 --bfile $geno --extract $snplist --make-bed --out $genoQC1"
pid1=$(qsubsh2.sh "$cmd1" 2 5G QC1 10:00:00 "-array=$chrs")

pid2=$(qsubsh2.sh "plink2 --bfile $genoQC1 --pheno $pheno --linear --out $assocOut" 2 5G assoc 10:00:00 "-array=$chrs -wait=$pid1")

qsubsh2.sh "Rscript test.R {TASK_ID}" 1 2G enrich 10:00:00 "-array=$chrs -wait=$pid2"
```

## What Changed Relative to Older qsubshcom-Style Usage

The calling interface stays the same, but `qsubsh2.sh` improves a few operational details:

- submission failures are logged clearly and return a non-zero exit code
- submission logs include the expanded backend submit command
- temporary job wrapper scripts are cleaned up automatically
- resource monitoring reports CPU time as summed CPU seconds instead of trying to add formatted time strings
- array task id handling is more robust across supported schedulers

## Issues

Open an issue in the repository if you hit a scheduler-specific incompatibility or want another example added to the README.
