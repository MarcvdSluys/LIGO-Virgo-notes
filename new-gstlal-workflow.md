- [Choose a singularity image/container for GstLAL](#org62ea097)
    - [Use a reference image/container](#org397da80)
    - [Set up a singularity image/container for GstLAL development](#orgc9e68ac)
  - [Set up the workflow/DAG](#org5829d31)
    - [Create a dir and download files](#orgcfe381b)
    - [Install the site-specific profiles](#org84589e0)
    - [Edit config.yml](#orgc79e67f)
    - [Create the workflow/DAG Makefile](#org13cc5ad)
    - [Set up a proxy if accessing non-public (GWOSC) data](#org8269cbb)
    - [Build the workflow/DAG file for submission](#orgf83465f)
      - [Possible issues](#orgc2a537c)
  - [Launch the workflow/DAG](#orge1208be)
    - [Possible issues](#org4414035)
  - [Generate the summary page](#orgd88a9c1)
  - [Resuming work in a new shell](#org210ad31)
  - [Submitting a rescue dag](#orgb84de52)
  - [Diagnosing and handling issues](#org74dfc31)
    - [Final nodes status](#org77219f7)
    - [List of errors](#orgc31fa45)

-   <span class="timestamp-wrapper"><span class="timestamp">[2021-11-23 Tue]</span></span>
-   <https://lscsoft.docs.ligo.org/gstlal/cbc_analysis.html>


<a id="org62ea097"></a>

# Choose a singularity image/container for GstLAL

-   A Singularity image or container contains a GNU/Linux system with (hopefully) the environment we need. The cluster environment is replaced with that of the container, except for e.g. your home directory.
-   Most important commands:
    
    ```bash
    singularity run <image>  # Replace the cluster environment with that of the container
    singularity exec <image> <command>  # Run <command> in the <image> environment, then return
    ```
-   See <https://sylabs.io/guides/latest/user-guide/> for more details.
-   You will have to choose between a **reference** (default, already setup) container and a **development** container.


<a id="org397da80"></a>

## Use a reference image/container

```bash
MYIMAGE="/cvmfs/singularity.opensciencegrid.org/lscsoft/gstlal:master"     # Default GstLAL master
MYIMAGE="/home/patrick.godwin/gstlal/offline/osg_small/gstlal-dev-210902"  # Patrick Godwin's version
```


<a id="orgc9e68ac"></a>

## Set up a singularity image/container for GstLAL development

-   <https://lscsoft.docs.ligo.org/gstlal/installation.html#singularity-container>
    
    ```bash
    # Pull a writable development container with GstLAL  installed in subdir gstlal-dev-container/:
    singularity build --sandbox --fix-perms gstlal-dev-container docker://containers.ligo.org/lscsoft/gstlal:master
    mkdir gstlal-dev-container/hdfs gstlal-dev-container/archive gstlal-dev-container/cvmfs  # They may be needed later
    
    MYIMAGE="$PWD/gstlal-dev-container"
    
    # Enter container (not needed?, test wether it works, exit with exit):
    # singularity run --writable $MYIMAGE  # gstlal-dev-container
    ```


<a id="org5829d31"></a>

# Set up the workflow/DAG

<https://lscsoft.docs.ligo.org/gstlal/cbc_analysis.html>


<a id="orgcfe381b"></a>

## Create a dir and download files

Download a default config file, mass model and template bank:

```bash
mkdir run-DAG-01 && cd run-DAG-01

curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/config.yml
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/mass_model/mass_model_small.h5
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/bank/gstlal_bank_small.xml.gz
```


<a id="org84589e0"></a>

## Install the site-specific profiles

-   Needed only **once** per user/cluster:
    
    ```bash
    singularity exec $MYIMAGE gstlal_grid_profile install  # Install profiles
    singularity exec $MYIMAGE gstlal_grid_profile list     # List installed profiles
    ```
-   This installs (and lists) `*.yml` files in `~/.config/gstlal/`.


<a id="orgc79e67f"></a>

## Edit config.yml

Set e.g.:

```yaml
start: 1187000000
stop: 1187100000

instruments: H1L1

data:
  template-bank: gstlal_bank_small.xml.gz

prior:
  mass-model: mass_model_small.h5

summary:
  webdir: ~/public_html/run-01
```

If your username doesn't match your LIGO albert.einstein name, you need to provide the latter:

```yaml
condor:
  accounting-group-user: albert.einstein
```

You may have to add or update the path to the singularity image. Make sure that this is the same as stored in `$MYIMAGE`.

```yaml
condor:
  singularity-image: /cvmfs/singularity.opensciencegrid.org/lscsoft/gstlal:master            
```

If running on an LDAS cluster rather than OSG, replace the profile:

```yaml
condor:
  profile: ldas
```

Do you need to change the data server? E.g.:

```yaml
source:
  data-find-server: ldr.ldas.cit:80
```

Reduced-order-model (ROM) waveforms present a faster version of a waveform model by storing a large number of precomputed waveforms and performing smart interpolation between them. The ROM files store these waveforms. You can think of these as the weights of a rudimental machine-learning model.

```yaml
directives:
  environment: '"LAL_DATA_PATH=/home/cbc/ROM_data SINGULARITY_BIND=/home/cbc/ROM_data"'
```

See <https://lscsoft.docs.ligo.org/gstlal/cbc_analysis.html#analysis-configuration> for more details on the configuration file.


<a id="org13cc5ad"></a>

## Create the workflow/DAG Makefile

```bash
singularity exec $MYIMAGE gstlal_inspiral_workflow init -c config.yml
# singularity exec $MYIMAGE gstlal_inspiral_workflow init -c config.yml -w injection  # Injection only
```

This creates a file called `Makefile`


<a id="org8269cbb"></a>

## Set up a proxy if accessing non-public (GWOSC) data

```bash
X509_USER_PROXY=/path/to/x509_proxy ligo-proxy-init -p albert.einstein
```

-   This asks for your LIGO password and creates a file called `x509_proxy` at the indicated location with certificates and a private key.
-   Note that this must be run **outside** the Singularity container.
-   The proxy is valid for a few hours **(CHECK: correct?)**
-   Note that you can use the `proxy-x509-create` alias to create `x509_proxy` in the current directory. <sup><a id="fnr.1" class="footref" href="#fn.1" role="doc-backlink">1</a></sup>

Edit `config.yml` and set the correct path to the proxy file:

```yaml
source:
  x509-proxy: /path/to/x509_proxy
```


<a id="orgf83465f"></a>

## Build the workflow/DAG file for submission

We need to select the whitening type using the environment variable `GSTLAL_FIR_WHITEN`. The value 0 sets the traditional acausal whitening filter, 1 enables causal whitening.

```bash
export GSTLAL_FIR_WHITEN=0  # Set to 0 or 1
singularity exec -B $TMPDIR $MYIMAGE make dag
```

-   This creates a list of files and subdirectories, amongst which Condor submission scripts (`*.sub`) and DAGMan files (`*.dag`).
-   Note: `$TMPDIR` is set when you login.


<a id="orgc2a537c"></a>

### Possible issues

1.  When running `make dag` (w/o singularity only?):
    -   ImportError: No module named \_lal
        -   goes away after trying a few times


<a id="orge1208be"></a>

# Launch the workflow/DAG

```bash
make launch  # Submit your DAG
condor_q     # Monitor your DAG
```

-   Note: run **outside** the Singularity image
-   `make launch` runs `condor_submit_dag` and should report something like `1 job(s) submitted to cluster xxx`
-   `condor_q` should show a few dozen to several hundred jobs, probably idle and perhaps running.
-   there should be a file called `*.dag.dagman.out` with status output. You can follow what's going on with e.g. `tail -f full_inspiral_dag.dag.dagman.out`
-   typical run time is in the order of hours, depending on your settings and cluster load.


<a id="org4414035"></a>

## Possible issues

1.  When running `make launch`
    
    ```bash
    ERROR: store_cred of LOCAL credential failed - The credmon did not process credentials within the timeout period
    ERROR: condor_submit failed; aborting.
    ```
    
    -   did you set up your proxy correctly?


<a id="orgd88a9c1"></a>

# Generate the summary page

```bash
make summary
# singularity exec -B $TMPDIR <image> make summary
```

-   The results from ldas Caltech will show up in <https://ldas-jobs.ligo.caltech.edu/~albert.einstein/>


<a id="org210ad31"></a>

# Resuming work in a new shell

If you log in in a new shell, the environment variables you had set will be gone. Hence, you will have resource one of the env files.

1.  before (re)building GstLAL:
    
    ```bash
    cd gstlal-deps
    source deps_env.sh
    cd -
    ```
2.  before (re)creating a (new) DAG:
    1.  Resource the GstLAL environment:
        
        ```bash
        cd gstlal-build
        source env.sh
        cd -
        ```
    2.  Resetup your proxy
        
        ```bash
        X509_USER_PROXY=/path/to/x509_proxy ligo-proxy-init -p albert.einstein
        ```


<a id="orgb84de52"></a>

# Submitting a rescue dag

-   the file `file.dag.dagman.out` or similar will show whether any of the nodes failed (near the end)
-   if this is the case, a `file.dag.rescueXXX` file will be created (where `XXX` is `001`, `002`, etc.)
-   the DAG can be resubmitted using the original DAG file: `condor_submit_dag file.dag`
-   Q: is this the correct way? Is this equivalent to redoing `make launch`?


<a id="org74dfc31"></a>

# Diagnosing and handling issues

-   The file `<name>_dag.dag.dagman.out` contains output of your job.


<a id="org77219f7"></a>

## Final nodes status

You can use e.g. dagman-out-final-status <sup><a id="fnr.2" class="footref" href="#fn.2" role="doc-backlink">2</a></sup> to grep the final status of the nodes:

```bash
$ dagman-out-final-status

full_inspiral_dag.dag.dagman.out:
12/01/21 14:09:45 Of 150 nodes total:
12/01/21 14:09:45  Done     Pre   Queued    Post   Ready   Un-Ready   Failed
12/01/21 14:09:45   ===     ===      ===     ===     ===        ===      ===
12/01/21 14:09:45    89       0        0       0       0         41       20
12/01/21 14:09:45 0 job proc(s) currently held
```

In this case, 20 nodes failed.


<a id="orgc31fa45"></a>

## List of errors

A list of errors, if any, can be shown using e.g. dagman-out-list-errors <sup><a id="fnr.3" class="footref" href="#fn.3" role="doc-backlink">3</a></sup>:

```bash
$ dagman-out-list-errors

full_inspiral_dag.dag.dagman.out:
12/01/21 14:09:45 ---------------------- Job ----------------------
12/01/21 14:09:45       Node Name: cluster_triggers_by_snr.00000
12/01/21 14:09:45            Noop: false
12/01/21 14:09:45          NodeID: 24
12/01/21 14:09:45     Node Status: STATUS_ERROR
12/01/21 14:09:45 Node return val: 1
12/01/21 14:09:45           Error: Job proc (245431516.0.0) failed with status 1 (after 3 node retries)
12/01/21 14:09:45 Job Submit File: cluster_triggers_by_snr.sub
12/01/21 14:09:45           Retry: 3
12/01/21 14:09:45  HTCondor Job ID: (245431516.0.0)
12/01/21 14:09:45 PARENTS: gstlal_inspiral.00000 WAITING: 0 CHILDREN: gstlal_inspiral_calc_likelihood.00000
etc...
```

Note that in this example, the Node name lists `cluster_triggers_by_snr.XXXXXX` as the culprit. In the subdirectory `logs/` you can find files called `cluster_triggers_by_snr_XXXXX-YYYYYYYYY-Z.err` containing more details (e.g. tracebacks):

```bash
less logs/cluster_triggers_by_snr_00000-245431516-0.err

Traceback (most recent call last):
...
```

## Footnotes

<sup><a id="fn.1" class="footnum" href="#fnr.1">1</a></sup> <https://github.com/MarcvdSluys/MyTerminalConfig/blob/master/bashrc_ligo>

<sup><a id="fn.2" class="footnum" href="#fnr.2">2</a></sup> <https://github.com/MarcvdSluys/MyTerminalConfig/blob/master/bin/dagman-out-final-status>

<sup><a id="fn.3" class="footnum" href="#fnr.3">3</a></sup> <https://github.com/MarcvdSluys/MyTerminalConfig/blob/master/bin/dagman-out-list-errors>
