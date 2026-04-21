# qsubsh2

A simple job submitter for PBS Pro, Torque, SGE and Slurm.

`qsubsh2` detects the cluster type automatically and submits jobs with the correct backend command without making you rewrite scheduler-specific headers such as `#PBS` or `#SBATCH`. It keeps the same calling style as classic [qsubshcom](https://github.com/zhilizheng/qsubshcom), but improves submission failure handling, submission logging, temporary script cleanup, and runtime resource monitoring.

`qsubsh2` has no extra dependency in a normal Linux cluster environment beyond the scheduler commands already installed by the system admin. Download `qsubsh2`, `chmod +x qsubsh2`, put it into your `PATH`, then it is ready to go.

Author: Zhili-inspired workflow, adapted `qsubsh2` version.  
License: follow your repository license.

## Installation

```bash
chmod +x qsubsh2

# if $HOME/bin is in your PATH
cp qsubsh2 ~/bin/qsubsh2

qsubsh2 "echo hello" 1 1G demo 00:01:00 ""
# if a job id is returned, installation is working
```

## Usage

```bash
# qsubsh2 command["one command |; two command"] num_CPU[1] total_memory[2G] task_name run_time[1:00:00] other_params
# job logs are in ./job_reports (*.log: stdout; *.err: stderr; *.mon: monitor)
# submit logs are in qsub.TIME.log or your custom -log name
qsubsh2 "echo 'hello world'" 1 1G helloTask 1:00:00 ""

# two or more commands in one job
qsubsh2 "echo 'hello world' ; echo 'hello2' |; echo 'hello3'" 1 1G helloTask 1:00:00 ""

#########################
## job dependency
job1=$(qsubsh2 "echo 'hello world 1'" 1 1G helloTask1 1:00:00 "")
job2=$(qsubsh2 "echo 'hello world 2'" 1 1G helloTask2 1:00:00 "")

# job3 will start after job1 and job2
command="echo hello world again"
job3=$(qsubsh2 "$command" 1 1G helloTask3 1:00:00 "-wait=$job1:$job2")

# array job, {TASK_ID} is the array index
qsubsh2 "echo 'hello world {TASK_ID}' > {TASK_ID}.log" 1 1G helloTask4 1:00:00 "-wait=$job3 -array=1-10"

#########################
## run existing scripts directly
qsubsh2 "./test_script.sh" 1 1G demo 1:00:00 ""
qsubsh2 "python3 test.py" 1 1G demoPy 1:00:00 ""
qsubsh2 "Rscript test.R --chr={TASK_ID}" 1 1G demoR 1:00:00 "-array=1-22"
```

If you already have scripts, you can run them as:

```bash
qsubsh2 "bash Your_script.sh" 2 4G your_task_name 10:00:00 ""
```

`qsubsh2` will go to the folder where you run the command, so `cd WORK_DIR` is usually not necessary. If you use job arrays inside your scripts, replace scheduler-specific variables such as `${PBS_ARRAY_INDEX}` or `${SLURM_ARRAY_TASK_ID}` with `${TASK_ID}` for cross-cluster compatibility.

## Example

```bash
chrs="1-22"
geno="test_chr{TASK_ID}"
pheno="pheno.txt"
snplist="chr{TASK_ID}.snplist"

genoQC1="extract_chr{TASK_ID}"
assocOut="test_assoc_chr{TASK_ID}"

# QC1 extract the SNPs out
cmd1="plink2 --bfile $geno --extract $snplist --make-bed --out $genoQC1"
pid1=$(qsubsh2 "$cmd1" 2 5G QC1 10:00:00 "-array=$chrs")

# wait for QC to finish
pid2=$(qsubsh2 "plink2 --bfile $genoQC1 --pheno $pheno --linear --out $assocOut" 2 5G assoc 10:00:00 "-array=$chrs -wait=$pid1")

# process the output
qsubsh2 "Rscript test.R {TASK_ID}" 1 2G enrich 10:00:00 "-array=$chrs -wait=$pid2"
```

If you want to pass special characters into the job script directly, they should be escaped, otherwise they may be interpreted on the head node instead of the working node.

```bash
temp_command="awk '{print \$1, \$2}' test.txt > test2.txt"
qsubsh2 "$temp_command" 1 1G test_awk 00:00:05 ""
```

## Memory

Total memory requested, not memory per CPU core. For example:

```bash
qsubsh2 "echo hello" 3 5G test 00:00:05 ""
```

This allocates 3 CPU cores and 5 GB memory in total.

For Torque and SGE, the requested memory may be rounded up if total memory divided by `num_CPU` is not an integer.

The memory should normally be written without `B`, such as `1G` or `100M`. Some schedulers do not accept `GB` or `MB`.

## Other_params

* `-wait=JOB1:JOB2:JOB3` wait these jobs to finish successfully before running
* `-array=1-100:2` create a job array that runs task `1 3 5 7 ... 99`
* `-log=log_name` write submit log to `log_name.YYYY.MM.DD.log`, default is `qsub.YYYY.MM.DD.log`
* `-ntype=node_type` specify node type for Torque-style clusters
* `-queue=queue_type` specify queue or partition
* `-acct=ACCOUNT_NAME` specify account, equivalent to PBS or Slurm `-A`
* `-ge=CLUSTER_ENGINE` specify cluster engine manually: `PBS`, `SGE`, `TOR`, `SLM`
* Any other scheduler-supported parameters are passed directly to `qsub` or `sbatch`, for example `-l host=host1:host2` or `--partition long`

Example:

```bash
"-wait=123:124:125 -array=1-2 -l host=host1:host2"
"--partition long --qos normal"
```

## Logs

Submit logs are written to `qsub.YYYY.MM.DD.log` by default, or to the file specified by `-log=`.

Job runtime logs are written to `./job_reports`:

* `*.log`: stdout
* `*.err`: stderr
* `*.mon`: monitor information

The real-time monitor records resource consumption every 10 seconds, including elapsed time, CPU seconds, `%CPU`, RSS, VSZ and `VmHWM`.

If submission fails, the failure is also written into the submit log and the script exits with a non-zero status.

## Update

Apr 21, 2026: Kept the original `qsubshcom` calling style, while improving submit failure logging, temporary file cleanup, and CPU time accounting in monitor logs.

## Issues

Report bugs in the repository issues tab.
