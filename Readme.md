# Run a script (DNAnexus Platform App)

## What does this app do?

This app provides a mechanism for running scripts as jobs on the DNA nexus 
platform.

## Use cases for this app

There are currently several ways scripts can be run on the DNA nexus platform:

(1) By launching an interactive workstation, e.g. using the JupyterLab, Posit 
    Workbench, or Cloud Workstation apps, then running the script directly from 
    that environment
(2) Using the Swiss Army Knife app to run a command that includes the uploaded 
    script.
(3) Developing a stand-alone applet runs the desired script.

In the first case, the user must log in to the interactive work station and run 
the script manually, which is not practical if you want to run many different 
jobs in parallel. In the second case, any outputs are only uploaded to project 
storage if the job completes successfully, and importantly it is not possible to
upload outputs to project storage directly to save partial progress for recovery 
from errors (or when AWS kills and restarts a low-priority instance). The third 
case requires significant additional overhead in terms of supporting code (the 
applet itself) and user-knowledge of applet development.

This app provides a mechanism for running scripts in way that is more similar to 
job submission on a high-performance computing cluster, allowing the user to 
spin up a workstation that automatically executes the user-provided script and 
shuts down when finished while providing the ability to upload output files to 
project storage regardless of job completion status.

The ability to write files to project storage while the job is running is 
particularly useful for minimizing costs of long running scripts, as it allows 
you to more effectively use the much cheaper low-priority setting by 
implementing checkpointing so that your job doesn't need to restart from scratch 
when [restarted by AWS](https://dnanexus.gitbook.io/uk-biobank-rap/working-on-the-research-analysis-platform/managing-jobs/managing-job-priority).

## Software provided by this app

For convenience this app provides a suite of pre-installed software beyond what 
is already provided by the base docker image used by all cloud workstations. 
This includes the latest java runtime environment, nextflow, plink, plink2, all 
the R packages that come pre-installed in the Posit Workbench app, plus a suite 
of additional R packages that take a long time to install (e.g. tidyverse).

## Key notes for app usage

The script file provided as input to this app will be downloaded from project 
storage to the home directory on the local cloud workstation, given executable 
permissions, then run directly as-is, e.g. as `./script.sh` if you have uploaded 
`path/to/script.sh` as your script. If a command string is also given as input 
to the optional cmd argument, then this command is run instead, giving a 
mechanism for executing non-bash scripts (e.g. giving `Rscript script.R` as the 
command if the upload script is an R script instead of bash script) and/or for 
giving additional named or positional arguments to script (e.g. 
`./script.sh --arg1 argument` as the cmd input).

Any other files on project storage that your script uses can either be accessed 
via the read-only mount point at `/mnt/project/` (mounted via `dxfuse`) similar 
to the Swiss Army Knife app, or can be downloaded from project storage in your 
script using `dx download`. Note for large files downloading first can often be 
faster than reading over the dxfuse mountpoint. 

Output files you want to keep will also need to be uploaded to project storage 
using `dx upload` in your script, as unlike the Swiss Army Knife app new files 
are not automatically uploaded on job completion. 

Unlike the interactive Posit Workbench and Jupyter Notebook apps, scripts run 
via this applet (and others like swiss-army-knife) are launched from a 
[temporary container project](https://documentation.dnanexus.com/developer/api/running-analyses/containers-for-execution). 
This means that when you run the DNAnexus command line tools like `dx upload` 
you need to provide the project ID otherwise you will be interacting with the 
temporary container instead of project storage. Determining the project ID can 
be done automatically at runtime via the `$DX_PROJECT_CONTEXT_ID` variable.

For convenience we have also developed a set of wrapper functions in R for 
uploading and downloading data while automatically bypassing project storage 
available via an R package [dxutils](https://github.com/sritchie73/dxutils).

The `dxutils` wrapper functions also extend `dx download` and `dx upload` in 
other various useful ways, including skipping already downloaded or uploaded 
files, waiting for incomplete files that are still being uploaded by other 
processes, replacing existing files on upload instead of keeping multiple 
copies, and detecting and automatically handling cases when the current job has 
been restarted by AWS mid-upload.

When running scripts on low-priority spot VMs that may be killed and restarted
at any time by AWS you should therefore always use the `dx_download()` and 
`dx_upload()` functions (and `dx_exists()` for file/folder checking).

The `dxutils` R package also provides a wrapper for `dx rm` that automatically
handles projects where you don't have delete permissions, by moving the files or
folders you want to delete to the trash/ folder.

This app also lets you set additional environment variables before running your 
script. This is particularly useful if you have scripts from high-performance 
computing environments where job scheduling was managed by slurm, as in 
conjunction with the `--batch-tsv` argument in `dx run` this creates 
cross-compatibility with scripts that have been set up to run as part of a slurm 
array job by setting the SLURM_ARRAY_TASK_ID environment variable before running 
your script.

## Example commands

```
# Run a script called job.sh on the default instance (mem1_ssd1_v2_x4)
dx run run_script -iscript='job.sh'

# Run a script called job.sh that takes additional arguments
dx run run_script -iscript='job.sh' -icmd='./job.sh --option --argument arg1'

# Run an Rscript called analysis.R
dx run run_script -iscript='analysis.R' -icmd='Rscript analysis.R'

# Set two environment variables before running job.sh
dx run run_script -iscript='job.sh' -ienv='SLURM_ARRAY_JOB_ID=1' -ienv='SLURM_CPUS_ON_NODE=4'

# Run a script designed to be run as an array job on a slurm HPC system - this
# will submit 22 identical jobs which differ only by the environment variable 
# passed to the run_script app; but you can use the --batch-tsv function to also
# vary other input arguments (e.g. to run jobs with different named or positional 
# arguments)
echo -e "Batch ID\tenv" > env_batch.tsv
for chr in {1..22}; do
  echo -e "chr_$chr\tSLURM_ARRAY_JOB_ID=$chr >> env_batch.tsv
done
dx run run_script -iscript='job.sh' --batch-tsv env_batch.tsv 
```

To learn how to change the instance type, enable remote ssh into the job, and
other job settings that are independent of this app, see `dx run --help`

## Installation

Setting up the applet involves a multi-step installation process as it involves
downloading and installing various tools and R packages so that they can be 
prebundled with the app (rather than installing every time the applet is 
launched)

First, download the code for the app on your local machine

```
git clone https://github.com/sritchie73/dxapplet-run_script.git
```

Second, run the scripts to download the linux binaries for dxfuse, plink, and 

```
cd dxapplet-run_script/resources/usr/src/
sh binary_installer.sh
cd ../../../
```

Third, build the app for the first time

```
dx build -f
```

Fourth, launch an interactive version of the applet:

```
touch dummy_script.sh
dx upload dummy_script.sh
rm dummy_script.sh
job_id=$(dx run run_script -iscript="dummy_script.sh" -icmd="while true; do sleep 10; done" --allow-ssh -y --brief)
```

Fifth, log into the interactive job:

```
# Use the job-id output by the previous step here, you may have to wait a bit
# first for the job to spin up, and some of the linux apps to install. If you
# get an error message saying 'Cannot resolve hostname or hostkey for <job-id>
# Please check your permissions and run settings.' after seeing the message 
# 'Resolving job hostname and SSH host key.....................................'
# this just means the 'dx ssh' command timed out while waiting for the instance
# to finish launching; so simply run the 'dx ssh' command below again and it 
# should log you in to an environment equivalent to 'dx run cloud_workstation'
dx ssh $job_id
```

Sixth, install the R packages from within the interactive job:

```
# This may take a couple of hours, at the end it will upload a file 
# Rpackages.tar.gz to your project storage
Rscript /usr/src/package_installer.R
```

Seven, back on your local machine, terminate the interactive job:

```
dx terminate $job_id
```

Eight, back on your local machine, download the installed R packages:

```
mkdir -p resources/usr/lib/R/site-library
dx download Rpackages.tar.gz -o resources/usr/lib/R/site-library/
```

Nine, finally rebuild the app again

```
dx build -f 
```

Ten, clean up temporary files on project storage created during the install process

```
dx rm Rpackages.tar.gz dummy_script.sh
```


