- [What problems are we solving?](#org100f2d8)
    - [Introducing: Singularity](#org04874d0)
    - [Yet Another Container Syndrome](#orgff81253)
    - [Important](#org2b9f537)
    - [Why Docker?](#org68bdcae)
    - [On Image Distribution](#org56e534b)
    - [Image Distribution](#orge75d30c)
    - [Integration with HTCondor](#org08ad94d)
    - [Example: all jobs into the container](#org3b0a347)
    - [Example: only on user request](#org55062a2)
    - [Example: Image based on OS name](#org7316ca9)
    - [Initial Singularity support for HTCondor](#orgac3d821)

-   Source: Brian Bockelman, HTCondor Week 2017: <https://research.cs.wisc.edu/htcondor/HTCondorWeek2017/presentations/WedBockelman_Singularity.pdf>


<a id="org100f2d8"></a>

# What problems are we solving?

1.  **Isolation:** We launch arbitrary user code (“payload”) that shouldn’t have access to our wrapper scripts (“pilot”). Specifically:
    1.  File isolation: pilot determines what files the payloads can read and write.
    2.  Process isolation: payload can only interact with (see, signal, trace) its own processes.
    3.  These are *simple* kinds of isolation. Others (e.g., kernel isolation, network isolation) are less important!
2.  **glexec replacement:** Retire our particularly problematic current solution to isolation. Niche and expensive.
3.  **Homogeneous / portable OS environments:** Make user OS environment as minimal and identical as possible!


<a id="org04874d0"></a>

# Introducing: Singularity

1.  Singularity is a container solution tailored for the HPC use case.
    1.  It allows for a portable of OS runtime environments.
    2.  It can provide isolation needed by our users.
2.  Simple isolation: Singularity does not do resource management (i.e., limiting memory use), leaving that to the batch system.
3.  Operations: No daemons, no UID switching; **no edits to config file needed**. “Install RPM and done.”
4.  Goal: User has no additional privileges by being inside container. E.g., disables all setuid binaries inside the container.

<http://singularity.lbl.gov>


<a id="orgff81253"></a>

# Yet Another Container Syndrome

-   But HTCondor already supports Docker! Why do we need Yet Another Container?
    1.  Singularity support works even if HTCondor runs as non-root (i.e., glideinWMS).
    2.  Singularity does not require any additional system services / daemons. Tradeoff: requires setuid.
    3.  Works inside Docker — important for sites that already invest heavily in Docker (like mine!).


<a id="org2b9f537"></a>

# Important

Singularity provides a path to non-setuid isolation And there was great rejoicing!


<a id="org68bdcae"></a>

# Why Docker?

1.  There remain a good number of reasons to use Docker universe:
    1.  Docker implements additional resource management and isolation mechanisms.
    2.  Built-in image distribution mechanism.
    3.  Wider acceptance / larger ecosystem / more mature.
2.  To each their own: pick the correct technology to fit your site.
3.  **Nebraska uses both:** Docker for site batch system, Singularity for pilots inside the batch system.


<a id="org56e534b"></a>

# On Image Distribution

-   Docker images are a list of layers, each a tarball.
    -   DockerHub limit is 10GB. In practice, ranges of 500MB (minimal image, caring users) to 4GB (large scientific organization) are common.
-   Singularity has three image formats:
    1.  Native format: raw filesystem image, loopback mounted. Large 10GB.
    2.  SquashFS-based compressed image. Slightly smaller than Docker (stays compressed on disk).
    3.  Simple chroot directory.
-   How does one deliver these to thousands of worker nodes?


<a id="orge75d30c"></a>

# Image Distribution

-   Observed several strategies in the wild:
    1.  Drop raw image onto shared file system.
    2.  Copy image files to worker node.
    3.  Synchronize chroot directory to CVMFS.

-   Tradeoffs to consider:
    1.  How much freedom will you give to users? Can they specify their own image? Are they restricted to a whitelist?
    2.  Use of cache (what is the working set size?). If user-specifies images, the working set size might be fairly unpredictable.
    3.  Scalability of distribution mechanism.
    4.  Does the full image get downloaded to the worker node?


<a id="org08ad94d"></a>

# Integration with HTCondor

-   Singularity availability and version advertised in ClassAd.
-   HTCondor will launch jobs inside Singularity based on a few condor<sub>startd</sub> configuration variables:
    1.  SINGULARITY<sub>JOB</sub>: If true, then launch job inside Singularity.
    2.  SINGULARITY<sub>IMAGE</sub><sub>EXPR</sub>: ClassAd expression; evaluated value is the path used for the Singularity image.
    3.  SINGULARITY<sub>TARGET</sub><sub>DIR</sub>: Location inside Singularity container where HTCondor working directory is mapped.
-   See <https://htcondor-wiki.cs.wisc.edu/index.cgi/tktview?tn=5828> for details. Examples follow. See [12](#orgac3d821) below
-   The details are a bit hidden under the cover; still experimenting with the best user interface.
    -   While base functionality is in 8.6.x, more UI work will occur in HTCondor 8.7.x.


<a id="org3b0a347"></a>

# Example: all jobs into the container

-   All config is controlled by the condor<sub>startd</sub>.
-   Example config:
    
    ```conf
    # Only set if Singularity is not in $PATH.  If found, the node should advertise HasSingularity=True
    # (or HAS_SINGULARITY=True, which is a DIFFERENT variable!) and become eligible for jobs requiring
    # Singularity.
    SINGULARITY = /cvmfs/oasis.opensciencegrid.org/mis/singularity/bin/singularity
    
    # Forces ALL jobs to run inside singularity.
    # SINGULARITY_JOB = true
    
    # Forces ALL jobs to use the CernVM-based image.
    # SINGULARITY_IMAGE_EXPR = "/cvmfs/cernvm-prod.cern.ch/cvm3"
    
    # Maps $_CONDOR_SCRATCH_DIR on the host to /srv inside the image.
    # MvdS: without this, our jobs on Ganymede started from $HOME rather than the HTCondor scratch dir
    #       ($_CONDOR_SCRATCH_DIR; typically /var/lib/condor/execute/dir_XXXXX or similar).
    #       As a consequence, it could not find input files (because they were transferred to the scratch
    #       dir).  Basically, this adds "--pwd /srv" to the singularity call, where /srv is a magical (and
    #       practically undocumented!) link to the HTCondor scratch dir.  With the setting below, the
    #       executable is started in /srv in the Singularity image, which is where the input files sit and
    #       whence the output files will be transferred back to the submit node.
    SINGULARITY_TARGET_DIR = /srv
    
    # Writable scratch directories inside the image. Auto-deleted after the job exits.
    MOUNT_UNDER_SCRATCH = /tmp, /var/tmp
    ```


<a id="org55062a2"></a>

# Example: only on user request

-   However, startd config variable can reference the user job using TARGET.
    
    ```conf
    SINGULARITY_JOB		= !isUndefined(TARGET.SingularityImage)
    SINGULARITY_IMAGE_EXPR 	= TARGET.SingularityImage
    ```

-   In this configuration, Singularity is only used if the user specifies an image in their submit file:
    
    ```conf
    +SingularityImage = "/cvmfs/cernvm-prod.cern.ch/cvm3"
    ```


<a id="org7316ca9"></a>

# Example: Image based on OS name

-   Startd config snippet:

```conf
SINGULARITY_JOB = \
(TARGET.DESIRED_OS isnt MY.OpSysAndVer) && \
((TARGET.DESIRED_OS is "CentOS6") || \
(TARGET.DESIRED_OS is "CentOS7"))

SINGULARITY_IMAGE_EXPR = \
(TARGET.DESIRED_OS is "CentOS6") ? \
”/cvmfs/singularity.opensciencegrid.org/library/centos:centos6” : \
”/cvmfs/singularity.opensciencegrid.org/library/centos:centos7”
```

-   User adds this to the job:

```conf
+DESIRED_OS="CentOS6"
```


<a id="orgac3d821"></a>

# Initial Singularity support for HTCondor

-   <https://htcondor-wiki.cs.wisc.edu/index.cgi/tktview?tn=5828>

Here is a summary of the changes:

-   condor<sub>startd</sub> detects the presence of singularity and tests to see if it is functional, then the machine ad advertises HasSingularity=True if it is found, along with version info. (just like Docker, Java, etc)
-   **SINGULARITY<sub>JOB</sub> = true:** forces the job to be run inside singularity. Evaluated in the context of the job (`TARGET`) and machine (`MY`) ad.
-   **SINGULARITY<sub>IMAGE</sub><sub>EXPR</sub>:** specifies the image. Evaluated in the context of the job (`TARGET`) and machine (`MY`) ad.
-   **SINGULARITY:** overrides the default path to the singularity binary.
-   **MOUNT<sub>UNDER</sub><sub>SCRATCH</sub>:** is respected.
-   Currently, specifying both `USE_PID_NAMESPACES=True` and `SINGULARITY_JOB` evaluating to `True` is not supported and **will result in failure to start jobs**. Need some improvement here?
-   HTCondor automatically bind mounts the $<sub>CONDOR</sub><sub>SCRATCH</sub><sub>DIR</sub> and IWD.

Additionally, singularity options can be set in `/etc/singularity/singularity.conf`
