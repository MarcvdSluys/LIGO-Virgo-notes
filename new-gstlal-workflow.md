- [Choose a singularity image/container for GstLAL](#org64b5d5b)
    - [Use a reference image/container](#orge99a99d)
    - [Set up a singularity image/container for GstLAL development](#org3a0fe65)
  - [Set up the workflow/DAG](#orgf001249)
    - [Create a dir and download files](#orgb668406)
    - [Install the site-specific profiles](#org37e4e7d)
    - [Edit `config.yml`](#orgd1c5b66)
    - [Create the workflow/DAG Makefile](#org49436f0)
    - [Set up a proxy if accessing non-public (GWOSC) data](#org0e84189)
    - [Build the workflow/DAG file for submission](#org3166188)
      - [Possible issues](#org04436bb)
  - [Launch the workflow/DAG](#org658f671)
    - [Possible issues](#orgc1712dd)
  - [Generate the summary page](#org3668ae6)
  - [Resuming work in a new shell or after a break](#orgf838dd1)
  - [Diagnosing and handling issues](#org7e46b2e)
    - [Final nodes status](#org0a9a05f)
    - [List of errors](#org7cdcdf5)
  - [Submitting a rescue dag](#orgdcd8361)
  - [A very short summary](#org1f2ac72)

-   <span class="timestamp-wrapper"><span class="timestamp">[2021-11-23 Tue]</span></span>
-   <https://lscsoft.docs.ligo.org/gstlal/cbc_analysis.html>


<a id="org64b5d5b"></a>

# Choose a singularity image/container for GstLAL

-   A Singularity image or container contains a GNU/Linux system with (hopefully) the environment we need. The cluster environment is replaced with that of the container, except for e.g. your home directory.
-   Most important commands:
    
    ```bash
    singularity run <image>  # Replace the cluster environment with that of the container
    singularity exec <image> <command>  # Run <command> in the <image> environment, then return
    ```
-   See <https://sylabs.io/guides/latest/user-guide/> for more details.
-   You will have to choose between a **reference** (default, already setup) container and a **development** container.


<a id="orge99a99d"></a>

## Use a reference image/container

```bash
MYIMAGE="/cvmfs/singularity.opensciencegrid.org/lscsoft/gstlal:master"     # Default GstLAL master
MYIMAGE="/home/patrick.godwin/gstlal/offline/osg_small/gstlal-dev-210902"  # Patrick Godwin's version
```


<a id="org3a0fe65"></a>

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


<a id="orgf001249"></a>

# Set up the workflow/DAG

<https://lscsoft.docs.ligo.org/gstlal/cbc_analysis.html>


<a id="orgb668406"></a>

## Create a dir and download files

Download a default config file, mass model and template bank:

```bash
mkdir run-DAG-01 && cd run-DAG-01

curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/config.yml
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/mass_model/mass_model_small.h5
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/bank/gstlal_bank_small.xml.gz
```


<a id="org37e4e7d"></a>

## Install the site-specific profiles

-   Needed only **once** per user/cluster:
    
    ```bash
    singularity exec $MYIMAGE gstlal_grid_profile install  # Install profiles
    singularity exec $MYIMAGE gstlal_grid_profile list     # List installed profiles
    ```
-   This installs (and lists) `*.yml` files in `~/.config/gstlal/`.


<a id="orgd1c5b66"></a>

## Edit `config.yml`

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


<a id="org49436f0"></a>

## Create the workflow/DAG Makefile

```bash
singularity exec $MYIMAGE gstlal_inspiral_workflow init -c config.yml
# singularity exec $MYIMAGE gstlal_inspiral_workflow init -c config.yml -w injection  # Injection only
```

This creates a file called `Makefile`


<a id="org0e84189"></a>

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


<a id="org3166188"></a>

## Build the workflow/DAG file for submission

We need to select the whitening type using the environment variable `GSTLAL_FIR_WHITEN`. The value 0 sets the traditional acausal whitening filter, 1 enables causal whitening.

```bash
export GSTLAL_FIR_WHITEN=0  # Set to 0 or 1
singularity exec -B $TMPDIR $MYIMAGE make dag
```

-   This creates a list of files and subdirectories, amongst which Condor submission scripts (`*.sub`) and DAGMan files (`*.dag`).
-   Note: `$TMPDIR` is set when you login.


<a id="org04436bb"></a>

### Possible issues

1.  When running `make dag` (w/o singularity only?):
    -   ImportError: No module named \_lal
        -   goes away after trying a few times


<a id="org658f671"></a>

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


<a id="orgc1712dd"></a>

## Possible issues

1.  When running `make launch`
    
    ```bash
    ERROR: store_cred of LOCAL credential failed - The credmon did not process credentials within the timeout period
    ERROR: condor_submit failed; aborting.
    ```
    
    -   did you set up your proxy correctly?


<a id="org3668ae6"></a>

# Generate the summary page

```bash
singularity exec -B $TMPDIR $MYIMAGE make summary
```

-   The results from ldas Caltech will show up in <https://ldas-jobs.ligo.caltech.edu/~albert.einstein/>


<a id="orgf838dd1"></a>

# Resuming work in a new shell or after a break

After you resume work, you may have to

1.  Redefine environment variables (for a new shell):
    
    ```bash
    MYIMAGE="/cvmfs/singularity.opensciencegrid.org/lscsoft/gstlal:master"     # Default GstLAL master
    # MYIMAGE="/home/patrick.godwin/gstlal/offline/osg_small/gstlal-dev-210902"  # Patrick Godwin's version
    
    export GSTLAL_FIR_WHITEN=0  # Set to 0 or 1
    ```
    
    or
    
    ```bash
    cd /path/to/gstlal-dev-container
    MYIMAGE="$PWD"
    ```
2.  Resetup your proxy
    
    ```bash
    X509_USER_PROXY=/path/to/x509_proxy ligo-proxy-init -p albert.einstein
    ```


<a id="org7e46b2e"></a>

# Diagnosing and handling issues

-   The file `<name>_dag.dag.dagman.out` contains output of your job.


<a id="org0a9a05f"></a>

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


<a id="org7cdcdf5"></a>

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


<a id="orgdcd8361"></a>

# Submitting a rescue dag

-   the file `file.dag.dagman.out` or similar will show whether any of the nodes failed (near the end)
-   if this is the case, a `file.dag.rescueXXX` file will be created (where `XXX` is `001`, `002`, etc.)
-   the DAG can be resubmitted using the original DAG file: `condor_submit_dag file.dag`
-   Q: is this the correct way? Is this equivalent to redoing `make launch`?


<a id="org1f2ac72"></a>

# A very short summary

```bash
MYIMAGE="/cvmfs/singularity.opensciencegrid.org/lscsoft/gstlal:master"     # Or one of the alternatives

mkdir run-DAG-01 && cd run-DAG-01

curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/config.yml
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/mass_model/mass_model_small.h5
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/bank/gstlal_bank_small.xml.gz

# Edit config.yml

singularity exec $MYIMAGE gstlal_inspiral_workflow init -c config.yml
proxy-x509-create

export GSTLAL_FIR_WHITEN=0  # Set to 0 or 1
singularity exec -B $TMPDIR $MYIMAGE make dag

make launch  # Submit your DAG
condor_q     # Monitor your DAG

# Wait...

singularity exec -B $TMPDIR $MYIMAGE make summary
```

## Footnotes

<sup><a id="fn.1" class="footnum" href="#fnr.1">1</a></sup> <https://github.com/MarcvdSluys/MyTerminalConfig/blob/master/bashrc_ligo>

<sup><a id="fn.2" class="footnum" href="#fnr.2">2</a></sup> <https://github.com/MarcvdSluys/MyTerminalConfig/blob/master/bin/dagman-out-final-status>

<sup><a id="fn.3" class="footnum" href="#fnr.3">3</a></sup> <https://github.com/MarcvdSluys/MyTerminalConfig/blob/master/bin/dagman-out-list-errors>
