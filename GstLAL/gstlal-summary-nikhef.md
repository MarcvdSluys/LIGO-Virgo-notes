- [Setup once](#org6313359)
  - [Workflow](#orgb52594b)

-   <span class="timestamp-wrapper"><span class="timestamp">[2022-02-09 Wed]</span></span>, for Ganymede


<a id="org6313359"></a>

# Setup once

-   Add these lines to your ~/.bashrc (or issue them for every shell):
    
    ```bash
    export SINGULARITY_BIND="/project,/data,/dcache,/tmp"  # Make these shares available in the Singularity containers
    export GSTLAL_IMG="/cvmfs/singularity.opensciencegrid.org/lscsoft/gstlal:master"     # Or one of the alternatives
    export GSTLAL_FIR_WHITEN=0  # Set to 0 or 1
    
    alias proxy-x509-create="X509_USER_PROXY=x509_proxy ligo-proxy-init -p <albert.einstein>"  # Replace <albert.einstein> with your user name
    alias cqw='watch -n 1 condor_q'  # Watch the Condor queue
    ```

-   Run once (per cluster/submit node):
    
    ```bash
    singularity exec $GSTLAL_IMG gstlal_grid_profile install  # Install profiles, once per cluster/submit node account
    
    # Add Nikhef (Ganymede) profile to GstLAL config:
    wget https://raw.githubusercontent.com/MarcvdSluys/LIGO-Virgo-files/master/GstLAL/config/nikhef.yml
    singularity exec $GSTLAL_IMG gstlal_grid_profile install nikhef.yml
    ```


<a id="orgb52594b"></a>

# Workflow

```bash
GSTLAL_IMG="/cvmfs/singularity.opensciencegrid.org/lscsoft/gstlal:master"     # Or one of the alternatives, if not set in your ~/.bashrc
export GSTLAL_FIR_WHITEN=0  # Set to 0 or 1 - if not done in your ~/.bashrc

# Create a working dir and get the input files fresh:
mkdir 01-run-dag && cd 01-run-dag
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/config.yml
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/mass_model/mass_model_small.h5
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/bank/gstlal_bank_small.xml.gz

# Or copy them from elsewhere:
mkdir 01-run-dag
cd 00-run-dag  # Template or previous run
cp config.yml mass_model_small.h5 gstlal_bank_small.xml.gz ../01-run-dag
cd ../01-run-dag

# Edit config.yml
#  under source:  data-find-server: datafind.ligo.org:443
#  under condor:  profile: nikhef

proxy-x509-create  # Will ask for your LIGO albert.einstein password and create x509_proxy (using the alias)

singularity exec $GSTLAL_IMG gstlal_inspiral_workflow init -c config.yml  # Create Makefile
singularity exec $GSTLAL_IMG make dag                                     # Create the DAG

make launch  &&  cqw                  # Submit your DAG and monitor it using the alias
tail -f full_inspiral.dag.dagman.out  # Monitor your DAG whilst running

# Wait...

singularity exec $GSTLAL_IMG make summary
```
