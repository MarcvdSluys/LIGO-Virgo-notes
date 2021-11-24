- [Set up and enter singularity container](#orge180353)
    - [Download and build the GstLAL dependencies](#org80103ae)
    - [Downloading and building GstLAL](#orgc78c9ca)
    - [Set up the workflow (DAG)](#org04287fd)
      - [Create a dir and download files](#org7d21637)
      - [Install the site-specific profiles](#org801f83a)
      - [Edit config.yml](#org8289a2d)
      - [Create the Makefile](#org7bbcddd)
      - [Set up a proxy if accessing non-public (GWOSC) data (**skipped for now**)](#orgcd6f1f3)
      - [Build the workflow/DAG file for submission](#org936b544)
        - [Debugging](#org6630330)
    - [Launch workflows](#org28ecf04)
    - [Generate summary page](#org1e4c658)

-   <span class="timestamp-wrapper"><span class="timestamp">[2021-11-23 Tue]</span></span>
-   <https://lscsoft.docs.ligo.org/gstlal/cbc_analysis.html>


<a id="orge180353"></a>

# Set up and enter singularity container

-   <https://lscsoft.docs.ligo.org/gstlal/installation.html#singularity-container>
    
    ```bash
    # Build writable container in subdir gstlal-dev-container/:
    singularity build --sandbox --fix-perms gstlal-dev-container docker://containers.ligo.org/lscsoft/gstlal:master
    mkdir gstlal-dev-container/hdfs gstlal-dev-container/archive gstlal-dev-container/cvmfs  # They may be needed later
    
    # Enter container (needed?):
    singularity run --writable gstlal-dev-container
    ```


<a id="org80103ae"></a>

# Download and build the GstLAL dependencies

```bash
mkdir gstlal-deps && cd gstlal-deps
wget https://git.ligo.org/lscsoft/gstlal/-/raw/master/gstlal-inspiral/share/post_O3/optimized/Makefile.ligosoftware_gcc_deps -O Makefile  # Download Makefile
less Makefile  # Verify the make instructions, `q` to exit

make deps_env.sh  # make deps_env.sh
ls -l  # check the result
source deps_env.sh  # source ('run') the script
time make 1> make.out 2> make.err &  # build the dependencies, ~16 min
tail -f make.out  # Follow installation (Ctrl-C to kill)
tail -f make.err  # Follow progress bit more relaxedly
```


<a id="orgc78c9ca"></a>

# Downloading and building GstLAL

Get GstLAL:

```bash
cd ..  # Get out of the gstlal-deps dir
git clone git@git.ligo.org:lscsoft/gstlal.git    # git/ssh - need key setup?  ~30s  
# git clone https://git.ligo.org/lscsoft/gstlal.git  # can't push over https?
```

-   the `git/ssh` version should use your ssh key if properly set up
-   it will ask your passphrase several times, so a key agent is already useful here

Build GstLAL:

```bash
mkdir gstlal-build && cd gstlal-build/
wget https://git.ligo.org/lscsoft/gstlal/-/raw/master/gstlal-inspiral/share/post_O3/optimized/Makefile.ligosoftware_gcc_gstlal -O Makefile  # Download Makefile
less Makefile  # Verify the make instructions
```

Edit `Makefile`:

1.  set the `DEPS_DIR:=` variable to the directory in which you built the dependencies, e.g. `../gstlal-deps`
2.  edit the `GSTLAL_GIT_BRANCH` variable if you want a different branch than `master`

Build GstLAL:

```bash
make env.sh
source env.sh
time make -f Makefile 1> make.out 2> make.err &  # ~25 min
```

-   this downloads and builds:
    1.  lalsuite: lal, lalframe, lalmetaio, lalsimulation, lalburst, lalinspiral, lalpulsar, lalinference, lalapps
    2.  gstlal: gstlal, gstlal-ugly, gstlal-burst, gstlal-calibration, gstlal-inspiral


<a id="org04287fd"></a>

# Set up the workflow (DAG)

<https://lscsoft.docs.ligo.org/gstlal/cbc_analysis.html>


<a id="org7d21637"></a>

## Create a dir and download files

Download a default config file, mass model and template bank:

```bash
cd ..  # Get out of the gstlal-build dir
mkdir run-DAG-01 && cd run-DAG-01

curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/config.yml
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/mass_model/mass_model_small.h5
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/bank/gstlal_bank_small.xml.gz
```


<a id="org801f83a"></a>

## Install the site-specific profiles

-   Needed only once per user/cluster

```bash
gstlal_grid_profile install
# singularity exec <image> gstlal_grid_profile install  # Same effect
gstlal_grid_profile list  # List installed profiles
```

This will install `*.yml` files in `~/.config/gstlal/`.


<a id="org8289a2d"></a>

## Edit config.yml

Set e.g.:

```conf
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

```conf
condor:
  accounting-group-user: albert.einstein
```

You may have to add or update the path to the singularity image:

```conf
condor:
  singularity-image: /cvmfs/singularity.opensciencegrid.org/lscsoft/gstlal:master            
```

If running on an LDAS cluster rather than OSG, replace the profile:

```conf
condor:
  profile: ldas
```

See <https://lscsoft.docs.ligo.org/gstlal/cbc_analysis.html#analysis-configuration> for more details on the configuration file.


<a id="org7bbcddd"></a>

## Create the Makefile

```bash
gstlal_inspiral_workflow init -c config.yml
# singularity exec <image> gstlal_inspiral_workflow init -c config.yml  # Doesn't work
# singularity exec <image> gstlal_inspiral_workflow init -c config.yml -w injection  # Injection only
```


<a id="orgcd6f1f3"></a>

## Set up a proxy if accessing non-public (GWOSC) data (**skipped for now**)

```bash
X509_USER_PROXY=/path/to/x509_proxy ligo-proxy-init -p albert.einstein
```

Edit `config.yml`:

```conf
source:
  x509-proxy: /path/to/x509_proxy
```


<a id="org936b544"></a>

## Build the workflow/DAG file for submission

```bash
make dag
# singularity exec -B $TMPDIR <image> make dag
```


<a id="org6630330"></a>

### Debugging

1.  make dag (w/o singularity):
    -   ImportError: No module named \_lal
    -   goes away after trying a few times
2.  make dag:
    -   You must set the environment variable GSTLAL<sub>FIR</sub><sub>WHITEN</sub> to either 0 or 1. 1 enables causal whitening. 0 is the traditional acausal whitening filter
    -   `export GSTLAL_FIR_WHITEN=0` Works! What does it mean?


<a id="org28ecf04"></a>

# Launch workflows

```bash
make launch  # Run condor_submit_dag
condor_q     # Monitor your dag
```


<a id="org1e4c658"></a>

# Generate summary page

```bash
make summary
# singularity exec -B $TMPDIR <image> make summary
```
