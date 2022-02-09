- [Setup once](#org630a73e)
    - [Workflow](#orgbe6710d)


<a id="org630a73e"></a>

# Setup once

```bash
GSTLAL_IMG="/cvmfs/singularity.opensciencegrid.org/lscsoft/gstlal:master"     # Or one of the alternatives - add to ~/.bashrc prefixed with export?
GSTLAL_FIR_WHITEN=0  # Set to 0 or 1 - add to ~/.bashrc prefixed with export?
singularity exec $GSTLAL_IMG gstlal_grid_profile install  # Install profiles, once per cluster account

# Add Nikhef (Ganymede) profile:
wget https://raw.githubusercontent.com/MarcvdSluys/LIGO-Virgo-files/master/GstLAL/config/nikhef.yml -O ~/.config/gstlal/nikhef.yml
```


<a id="orgbe6710d"></a>

# Workflow

-   for Ganymede

```bash
# Choose one (set the default in your ~/.bash_profile ?):
GSTLAL_IMG="/cvmfs/singularity.opensciencegrid.org/lscsoft/gstlal:master"     # Or one of the alternatives
cd /path/to/gstlal-dev-container && GSTLAL_IMG="$PWD" && cd -

mkdir 001-run-dag && cd 001-run-dag

# Get them fresh:
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/config.yml
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/mass_model/mass_model_small.h5
curl -O https://git.ligo.org/gstlal/offline-configuration/-/raw/main/bns-small/bank/gstlal_bank_small.xml.gz
# Or copy them from elsewhere:
cd ../000-run-dag  # Template or previous run
cp config.yml mass_model_small.h5 gstlal_bank_small.xml.gz $OLDPWD
cd -

# Edit config.yml
# Nikhef:
#   under source:  data-find-server: datafind.ligo.org:443
#   under condor:  profile: ldas

singularity exec -B $PWD $GSTLAL_IMG gstlal_inspiral_workflow init -c config.yml  # Create Makefile - Nikhef

proxy-x509-create  # Will ask for your LIGO albert.einstein password

# export GSTLAL_FIR_WHITEN=0  # Set to 0 or 1 - if not done in your ~/.bashrc

singularity exec -B $TMPDIR,$PWD $GSTLAL_IMG make dag  # Create the DAG - Nikhef

gstlal_nikhef_fix_condor_submission_files  # Nikhef

make launch  &&  watch -n 1 condor_q  # Submit your DAG and monitor it
tail -f full_inspiral.dag.dagman.out  # Monitor your DAG whilst running

# Wait...

singularity exec -B $TMPDIR $GSTLAL_IMG make summary
```
