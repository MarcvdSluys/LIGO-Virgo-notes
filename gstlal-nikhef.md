- [Do only once](#org6b64524)
    - [Workflow](#org75162bd)


<a id="org6b64524"></a>

# Do only once

```bash
singularity exec $MYIMAGE gstlal_grid_profile install  # Install profiles, once per cluster account
# Add "export GSTLAL_FIR_WHITEN=0  # Set to 0 or 1" to your ~/.bashrc?
```


<a id="org75162bd"></a>

# Workflow

```bash
# Choose one:
MYIMAGE="/cvmfs/singularity.opensciencegrid.org/lscsoft/gstlal:master"     # Or one of the alternatives
MYIMAGE="/home/patrick.godwin/gstlal/offline/osg_small/gstlal-dev-210902"  # Patrick Godwin's version - doesn't work?
cd /path/to/gstlal-dev-container && MYIMAGE="$PWD" && cd -

mkdir run-dag-01 && cd run-dag-01

# Get them fresh:
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/config.yml
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/mass_model/mass_model_small.h5
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/bank/gstlal_bank_small.xml.gz
# Or copy them from elsewhere:
cd ../run-dag-00  # Previous run
cp config.yml mass_model_small.h5 gstlal_bank_small.xml.gz $OLDPWD
cd -

# Edit config.yml

singularity exec $MYIMAGE gstlal_inspiral_workflow init -c config.yml  # Create Makefile
singularity exec -B $PWD $MYIMAGE gstlal_inspiral_workflow init -c config.yml  # Create Makefile - Nikhef

proxy-x509-create  # Will ask for your LIGO albert.einstein password

# export GSTLAL_FIR_WHITEN=0  # Set to 0 or 1 - if not done in your ~/.bashrc

singularity exec -B $TMPDIR $MYIMAGE make dag  # Create the DAG
singularity exec -B $TMPDIR,$PWD $MYIMAGE make dag  # Create the DAG - Nikhef

make launch  # Submit your DAG
condor_q     # Monitor your DAG
tail -f full_inspiral.dag.dagman.out  # Monitor your DAG whilst running

# Wait...

singularity exec -B $TMPDIR $MYIMAGE make summary
```
