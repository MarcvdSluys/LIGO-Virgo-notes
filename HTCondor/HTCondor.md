- [Lingo](#org429674f)
  - [Condor submit script](#org42bc396)
    - [Condor submit-script parameters/commands](#org7e7a2d3)
    - [Python in Condor](#orgfd8b9cc)
  - [Condor DAGMan](#org1a2b059)
    - [Debugging your DAG](#org3645791)
  - [Monitor your DAGs/runs/the cluster](#orgbea4956)
  - [Cluster/node/cpu info](#org5e50b0f)
  - [OSG/HTCondor troubleshooting](#orgded42b2)
    - [Condor structure](#org26a02a8)
    - [Possible issues](#orgb80b290)
    - [What are my jobs up to?](#orgaa78569)
    - [Why are my jobs idle?](#org433ec21)
    - [Why are my jobs still running?](#orgb357bfa)
    - [Why are my jobs held?](#org014b665)
    - [My jobs completed, but the output is wrong](#org9c7be28)
    - [Troubleshooting DAGs](#org920243d)

-   <https://htcondor.readthedocs.io/en/latest/users-manual/>


<a id="org429674f"></a>

# Lingo

-   **Shadow :** submit node
-   **Starter:** compute node


<a id="org42bc396"></a>

# Condor submit script

-   <https://htcondor.readthedocs.io/en/latest/users-manual/submitting-a-job.html>

```bash
# job.sub -- submit a simple bash job
executable              = job.sh
log                     = job.log
output                  = job.out
error                   = job.err
should_transfer_files   = Yes
when_to_transfer_output = ON_EXIT

queue
```

-   submission: `condor_submit job.sub`


<a id="org7e7a2d3"></a>

## Condor submit-script parameters/commands

-   **`universe`:** use vanilla?
-   **`executable`:** path/name of executable
-   **`arguments`:** cli options and arguments for executable
-   **`periodic_release`:** 

-   **`request_cpus`:** number of cpus needed
-   **`request_memory`:** amount of memory needed
-   **`want_graceful_removal`:** 

-   **`kill_sig`:** 

-   **`accounting_group`:** set for LIGO jobs
-   **`accounting_group_user`:** set for LIGO jobs
-   **`environment`:** set environment variables for job: e.g. `environment = "DIR=/path/to/dir"`
-   **`+SingularityImage`:** path to Singularity image (also acts as switch between using S. or not)
-   **`transfer_executable`:** transfer the executable from the local dir (`YES/NO`)
-   **`getenv`:** 

-   **`x509userproxy`:** path/name of the `x509_proxy` file
-   **`use_x509userproxy`:** `True` for LIGO
-   **`requirements`:** compute node CPU family, processor capabilities, availability of Singularity, &#x2026;
-   **`should_transfer_files`:** transfer input files to compute node and/or output files back to submit node
-   **`when_to_transfer_output`:** `ON_EXIT`, &#x2026;, `ON_SUCCESS` (latter from ~v8.9.7 onwards)
-   **`success_exit_code`:** 

-   **`preserve_relative_paths`:** >v8.8
-   **`transfer_input_files`:** comma-separated list of file names
-   **`transfer_output_files`:** comma-separated list of file names
-   **`transfer_output_remaps`:** doubly-quoted, semicolon-separated list of remaps(?)
-   **`output`:** file that should contain stdout
-   **`error`:** file that should contain stderr
-   **`notification`:** 


-   **`queue`:** queue command (ends the settings)?


<a id="orgfd8b9cc"></a>

## Python in Condor

-   **Duncan McL:** If you are running a python script in a condor job, (I recommend) you should always use as the executable either the **absolute path** of a python interpreter, or the **absolute path** of a python script that includes a shebang line that itself includes the **absolute path** of a python interpreter - anything else is just asking for environment troubles.


<a id="org1a2b059"></a>

# Condor DAGMan

-   <https://htcondor.readthedocs.io/en/latest/users-manual/dagman-workflows.html>


<a id="org3645791"></a>

## Debugging your DAG

-   See also [6](#orgded42b2)

In order to debug your DAG, you may want to rerun only the single job that failed, rather than the whole DAG. Steps could include:

1.  Start from the original HTCondor submit file: `cp <job>.sub <job>_test.sub1` (the extension ensures that it doesn't get deleted by e.g. `make clean` in GstLAL)
2.  Edit `<job>_test.sub1` to:
    1.  replace the executable with `<job>_test.sh`
    2.  (remove the arguments?)
    3.  replace in the variables used with their values:
        -   variables in the Condor submit files `*.sub` are (probably) declared in the dagman `*.dag` files.
            -   check for lines like `VARS title(?) var1="value 1" var2="value 2"`
            -   when using them in `<job>_test.sub1` they translate to:
                
                ```conf
                var1=value 1
                var2=value 2
                ```
                
                (i.e. without the quotes, even if `value i` contains spaces).
        -   especially check the `*_transfer_*` statements, with input/output files that should be **transferred** from the submit node to the compute node and back.
    4.  ensure that `transfer_executable = True` (`<job>_test.sh` should be transferred to the compute node).
    5.  ensure stdout and stderr are redirected into files (typically `<job>_test.out/.err`) for examination later.
3.  Create the script `<job>_test.sh`, which should:
    1.  run your executable with the proper arguments
    2.  perhaps feature some `ls` commands (e.g. `ls -lrt`) to check whether all necessary executable and input/output files are present in the Condor scratch directory before/after running your executable
    3.  other commands like `cat` to check the content of any files or `env` or `echo $VAR` to check the environment on the compte node.
4.  Submit your test job with `condor_submit <job>_test.sub1` and wait for it to finish or be held (in which case you may want to kill it with `condor_rm <jobnr>`).
5.  Examine the `*.out` and/or `*.err` files for issues. Note that
    -   **transfer problems** mean the copying of executable/input/output files between the submit and compute nodes
        -   the submit node is called **SHADOW**
        -   the compute node is called **STARTER**


<a id="orgbea4956"></a>

# Monitor your DAGs/runs/the cluster

-   <https://htcondor.readthedocs.io/en/latest/man-pages/condor_q.html>

```bash
condor_q  # Monitor your DAGs/jobs
condor_q -allusers  # All users
condor_q -analyze 846.0  # Details on job
condor_q -better 846.0  # More details on job - same as -better-analyze

condor_q -unmatchable  # List with some details on jobs that do not match any machines

condor_rm <job_id>  # Remove job or DAG <job_id>
condor_rm <user>    # Remove all <user>'s jobs
```

-   Note: if a DAG is removed, it may split up into a list of individual jobs for a number of seconds before disappearing.
-   set aliases cq, cqa, cqb, cqw, crm, &#x2026;


<a id="org5e50b0f"></a>

# Cluster/node/cpu info

```bash
condor_status  # What is each node doing?  (Claimed ~ Busy;  Unclaimed ~ Idle)
condor_status -total  # Just show totals  (see my script condor_status_totals)
condor_status -long   # Detailed info on ALL cpus
```


<a id="orgded42b2"></a>

# OSG/HTCondor troubleshooting

-   Source: OSG user school 2017: <https://opensciencegrid.org/user-school-2017/materials/day2/files/osgus17-day2-part4-troubleshooting.pdf>


<a id="org26a02a8"></a>

## Condor structure

![img](Figs/gw-clusters-HTCondor-structure.png)


<a id="orgb80b290"></a>

## Possible issues

1.  `ERROR: Can't open "/cloud/login/blin/school/input_data" with flags 00 (No such file or directory)`
    -   no executable specified?
2.  `ERROR: I don't know about the 'vanila' universe`
    -   typo?
3.  `ERROR: Executable file /bin/slep does not exist`
    -   typo?
4.  Note: Errors appears in `*dagman.out` files instead of STDOUT or STDERR


<a id="orgaa78569"></a>

## What are my jobs up to?

```bash
condor_q -help status
```

Job status codes:

-   **I:** `IDLE`
-   **R:** `RUNNING`
-   **X:** `REMOVED`
-   **C:** `COMPLETED`
-   **H:** `HELD`
-   **>:** `TRANSFERRING_OUTPUT`
-   **S:** `SUSPENDED`


<a id="org433ec21"></a>

## Why are my jobs idle?

```bash
condor_q -better 29486
```

-   shows desired and (un)matched requirements


<a id="orgb357bfa"></a>

## Why are my jobs still running?

-   Solution: Use `condor_ssh_to_job <job ID>` to open an SSH session to the worker node running your job.
    -   Non-OSG jobs only!


<a id="org014b665"></a>

## Why are my jobs held?

```bash
condor_q -held
```

-   Condor couldn’t run your executable, e.g. because:
    1.  `Failed to execute '/path/to/executable': (errno=13: 'Permission denied')`
        -   no executable permission?
    2.  `Failed to execute '/path/to/executable': (errno=8: 'Exec format error')`
        -   no shebang?
        -   use absolute paths (also in shebang?)
    3.  `Failed to initialize user log to /path/to/test-000.log or /path/to/./test.dag.nodes.log`
        -   path does not exist or has no write permissions?
    4.  `Failed to execute '/cloud/login/blin/school/sleep.sh': invalid interpreter (/bin/bash) specified on first line of script (errno=2: 'No such file or directory')`
        -   There may be carriage returns (`^M`) in your executable (use dos2unix)
    5.  `Job has gone over memory limit of 1 megabytes. Peak usage: 1 megabytes`
        -   Request more resources:
            1.  Solution 1: Edit your jobs on the fly:
                
                ```bash
                condor_qedit <job ID> <resource> <value>  # e.g.:
                condor_qedit <job ID> RequestMemory <mem_in_MB>
                condor_qedit -const ‘JobStatus =?= 5’ RequestDisk <disk_in_KiB>
                condor_qedit -const ‘Owner =?= “blin”’ RequestCpus <CPUs>
                
                # Then:
                condor_release <job ID>
                ```
            2.  Solution 2: Remove job, fix submit file and resubmit job:
                
                ```bash
                condor_rm <job ID>  # Add request_disk, request_mem, or request_cpus to your submit file
                condor_submit <submit file>  
                ```
    6.  STARTER at `<address>` failed to send file(s) to `<address>`: error reading from `/var/lib/condor/execute/dir_xxx/bar`: (errno 2) No such file or directory; SHADOW failed to receive file(s) from `<128.104.100.52:10507>`
        
        -   ISSUE: Your job did not create the files that you specified in `transfer_output_files`.
        -   SOLUTION:
            1.  Check for typos in `transfer_output_files`, job runtime
            2.  add debugging information to your code
        -   Note:
        
        ![img](Figs/gw-clusters-HTCondor-shadow-starter.png)


<a id="org9c7be28"></a>

## My jobs completed, but the output is wrong

1.  Check `*.log` files for return codes or unexpected behavior: short runtimes, using too many or too few resources
2.  Check `*.err` and `*.out` for error messages.
3.  Submit an interactive job: `condor_submit -i <submit file>` and run the executable manually.
    1.  If it succeeds, does your submit file have the correct args? If yes, try adding `GET_ENV=True` to your submit file.
    2.  If it fails, there is an issue with your code or your invocation!


<a id="org920243d"></a>

## Troubleshooting DAGs

1.  Check `*.rescue*` files (which DAG nodes failed)
2.  Check `*.dagman.out` (errors with job submission)
3.  Check `*.nodes.log` (return codes, PRE/POST script failures).
4.  If PRE/POST scripts failed, run them manually to see where they failed.
