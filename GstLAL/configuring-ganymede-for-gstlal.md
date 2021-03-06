
# Table of Contents

1.  [Add Singularity dir to PATH](#org41fb940)
2.  [Run in dcache](#org1015893)
3.  [Define TMPDIR](#orgb1bedca)
4.  [Bind directories to Singularity](#orgbca907a)
5.  [Install LIGO software packages](#org0f00d76)
6.  [DAG can be submitted, but first two individual jobs fail](#orgd90c712)
7.  [Match HTCondor ClassAds with job requirements](#orgfc4052d)
8.  [Ensure jobs run in a Singularity container](#org326528d)
    1.  [Issue: jobs run, but crash immediately](#org3c38bf4)
    2.  [Diagnose the issue](#org5d5cb1c)
9.  [Use correct data server](#orgacaba89)
10. [Ensure data directory is mounted and bound](#orgfbb1c2e)
11. [Upgrade HTCondor](#org5fd2734)



<a id="org41fb940"></a>

# Add Singularity dir to PATH

Add `/cvmfs/oasis.opensciencegrid.org/mis/singularity/bin` to `PATH` for singularity binary, e.g. in
`~/.bash_profile`:

    PATH="$PATH:/cvmfs/oasis.opensciencegrid.org/mis/singularity/bin"


<a id="org1015893"></a>

# Run in dcache

-   run in `/dcache/gravwav/`
-   for high-throughput output from the jobs and long-term storage
-   simply do `mkdir /project/gravwav/$USER`, even if you cannot see the parent dir, and it will appear
    (automount).
-   Users must be added to `gravwav` group?


<a id="orgb1bedca"></a>

# Define TMPDIR

-   Must define `$TMPDIR`, e.g. in `~/.bashrc`:

    export TMPDIR="/tmp"


<a id="orgbca907a"></a>

# Bind directories to Singularity

-   `/dcache` and `$TMPDIR` are not available in your singularity container by default
    -   you need to bind them using `-B`
-   when launching your job from its working directory, you can use `$PWD`: `-B $TMPDIR,$PWD`


<a id="org0f00d76"></a>

# Install LIGO software packages

-   `proxy-x509-create` == `X509_USER_PROXY=x509_proxy ligo-proxy-init -p marc.vandersluys`:
-   DONE: installed many lscsoft/ligo packages e.g. ligo-proxy-utils, lscsoft-all, -frame, -gds, -gstlal(-dev),
    -lalsuite-dev, -ldas-tools(-dev), -ligotools, &#x2026;
-   NOTE: ligo username != Nikhef user name
    -   DONE: adapt in bash


<a id="orgd90c712"></a>

# DAG can be submitted, but first two individual jobs fail

-   from log:
    
        01/21/22 17:18:09 Submitting HTCondor Node split_injections.00000 job(s)...
        01/21/22 17:18:09 Adding a DAGMan workflow log /user/sluijsm/GstLAL/FirstLight/run-dag-02/./full_inspiral_dag.dag.nodes.log
        01/21/22 17:18:09 Masking the events recorded in the DAGMAN workflow log
        01/21/22 17:18:09 Mask for workflow log is 0,1,2,4,5,7,9,10,11,12,13,16,17,24,27,35,36
        01/21/22 17:18:09 submitting: /usr/bin/condor_submit -a dag_node_name' '=' 'split_injections.00000 -a +DAGManJobId' '=' '761 -a DAGManJobId' '=' '761 -batch-name full_inspiral_dag.dag+761 -a submit_event_notes' '=' 'DAG' 'Node:' 'split_injections.00000 -a dagman_log' '=' '/user/sluijsm/GstLAL/FirstLight/run-dag-02/./full_inspiral_dag.dag.nodes.log -a +DAGManNodesMask' '=' '"0,1,2,4,5,7,9,10,11,12,13,16,17,24,27,35,36" -a nodename' '=' 'split_injections_00000 -a nsplit' '=' '--nsplit' '1 -a usertag' '=' '--usertag' 'BNS -a injection_file' '=' 'bns_injections.xml -a input_injection_file' '=' 'bns_injections.xml -a output_path' '=' '--output-path' 'filter/split_injections -a output_output_path' '=' 'filter/split_injections -a output_split_injections' '=' 'filter/split_injections/H1K1L1V1-0000_GSTLAL_SPLIT_INJECTIONS_BNS-0-0.xml -a DAG_STATUS' '=' '2 -a FAILED_COUNT' '=' '1 -a +KeepClaimIdle' '=' '20 -a notification' '=' 'never -a +DAGParentNodeNames' '=' '"" split_injections.sub
        01/21/22 17:18:09 From submit: Submitting job(s)
        01/21/22 17:18:09 From submit: ERROR: invalid value ("ON_SUCCESS") for WhenToTransferOutput. Please either 
        01/21/22 17:18:09 From submit: specify "ON_EXIT", or "ON_EXIT_OR_EVICT" and try again. 
        01/21/22 17:18:09 failed while reading from pipe.
        01/21/22 17:18:09 Read so far: Submitting job(s)ERROR: invalid value ("ON_SUCCESS") for WhenToTransferOutput. Please either specify "ON_EXIT", or "ON_EXIT_OR_EVICT" and try again. 
        01/21/22 17:18:09 ERROR: submit attempt failed
        01/21/22 17:18:09 submit command was: /usr/bin/condor_submit -a dag_node_name' '=' 'split_injections.00000 -a +DAGManJobId' '=' '761 -a DAGManJobId' '=' '761 -batch-name full_inspiral_dag.dag+761 -a submit_event_notes' '=' 'DAG' 'Node:' 'split_injections.00000 -a dagman_log' '=' '/user/sluijsm/GstLAL/FirstLight/run-dag-02/./full_inspiral_dag.dag.nodes.log -a +DAGManNodesMask' '=' '"0,1,2,4,5,7,9,10,11,12,13,16,17,24,27,35,36" -a nodename' '=' 'split_injections_00000 -a nsplit' '=' '--nsplit' '1 -a usertag' '=' '--usertag' 'BNS -a injection_file' '=' 'bns_injections.xml -a input_injection_file' '=' 'bns_injections.xml -a output_path' '=' '--output-path' 'filter/split_injections -a output_output_path' '=' 'filter/split_injections -a output_split_injections' '=' 'filter/split_injections/H1K1L1V1-0000_GSTLAL_SPLIT_INJECTIONS_BNS-0-0.xml -a DAG_STATUS' '=' '2 -a FAILED_COUNT' '=' '1 -a +KeepClaimIdle' '=' '20 -a notification' '=' 'never -a +DAGParentNodeNames' '=' '"" split_injections.sub
        01/21/22 17:18:09 Job submit try 3/6 failed, will try again in >= 4 seconds.
-   i.e., ERROR: invalid value ("ON<sub>SUCCESS</sub>") for WhenToTransferOutput. Please either specify "ON<sub>EXIT</sub>", or
    "ON<sub>EXIT</sub><sub>OR</sub><sub>EVICT</sub>" and try again.
    -   Duncan McLeod: `ON_SUCCESS` introduced in HTCondor v8.9.7
    -   WORKAROUND: run-dag-02 for fun: `sed -i 's|ON_SUCCESS|ON_EXIT|g' *.sub` and resubmit
    -   later solution: upgrade HTCondor to 8.9.13


<a id="orgfc4052d"></a>

# Match HTCondor ClassAds with job requirements

-   ISSUE: DAG is submitted, jobs are no longer killed, but do not run either (for days)
-   ISSUE: Condor CPU requirements cannot be met, hence Condor is waiting for a CPU with the proper
    requirements to pop up, which neven happens
    -   WORKAROUND: strip them: `sed -i '/requirements/d' *.sub`
        -   between `make dag` and `make launch`
    -   RESULT:
-   `condor_q -better-analyze`:
    
        condor_q -better-analyze 896
        -- Schedd: visar.nikhef.nl : <145.107.7.239:9618?...
        The Requirements expression for job 896.000 is
        
            (has_avx2 && (CpuFamily is 6) && (HAS_SINGULARITY is true)) && (TARGET.Arch == "X86_64") && (TARGET.OpSys == "LINUX") && (TARGET.Disk >= RequestDisk) && (TARGET.Memory >= RequestMemory) && (TARGET.Cpus >= RequestCpus) &&
            (TARGET.HasFileTransfer)
        
        Job 896.000 defines the following attributes:
        
            DiskUsage = 5
            RequestCpus = 2
            RequestDisk = DiskUsage
            RequestMemory = 2000
        
        The Requirements expression for job 896.000 reduces to these conditions:
        
             Slots
        Step    Matched  Condition
        -----  --------  ---------
        [1]           0  CpuFamily is 6
        [3]           0  HAS_SINGULARITY is true
        [13]          0  TARGET.Cpus >= RequestCpus
        
        No successful match recorded.
-   ISSUES:
    1.  job requires CPU of family 6, which we don't have
        -   this turns out to come from the `ldas.yml` config
        -   SOLUTION: create `nikhef.yml` for CpuFamily 23
            
                singularity exec $GSTLAL_IMG gstlal_grid_profile install  # Install GstLAL cluster profiles
                
                # In addition: add the Nikhef (Ganymede) GstLAL profile:
                wget https://raw.githubusercontent.com/MarcvdSluys/LIGO-Virgo-files/master/GstLAL/config/nikhef.yml
                singularity exec $GSTLAL_IMG gstlal_grid_profile install nikhef.yml
    
    2.  job requires Singularity, but HAS<sub>SINGULARITY</sub> is false
        -   added to HTCondor/signularity config:
            
                HAS_SINGULARITY = HasSingularity
                STARTD_ATTRS = $(STARTD_ATTRS) HAS_SINGULARITY
            
            -   The second line solves the additional issue that the GstLAL `*.sub` files require
                `HAS_SINGULARITY=?=True` whereas our HTCondor advertises `HasSingularity = true`
    
    3.  job requires >1 CPU, Condor advertises as having 384 machines with 1 cpu each
        -   SOLUTION: mend this in the cluster config (somehow)


<a id="org326528d"></a>

# Ensure jobs run in a Singularity container


<a id="org3c38bf4"></a>

## Issue: jobs run, but crash immediately

-   `logs/split_injections_00000-847-0.err`:
    
        Traceback (most recent call last):
          File "/usr/bin/gstlal_injsplitter", line 83, in <module>
            cvs_entry_time=strftime('%Y/%m/%d %H:%M:%S'))
          File "/usr/lib64/python3.6/site-packages/ligo/lw/utils/process.py", line 109, in register_to_xmldoc
            process = proctable.RowType.initialized(program = program, process_id = proctable.get_next_id(), **kwargs)
          File "/usr/lib64/python3.6/site-packages/ligo/lw/lsctables.py", line 562, in initialized
            cvs_entry_time = lal.UTCToGPS(time.strptime(cvs_entry_time, "%Y-%m-%d %H:%M:%S +0000")) if cvs_entry_time is not None else None,
          File "/usr/lib64/python3.6/_strptime.py", line 559, in _strptime_time
            tt = _strptime(data_string, format)[0]
          File "/usr/lib64/python3.6/_strptime.py", line 362, in _strptime
            (data_string, format))
        ValueError: time data '2022/01/26 16:47:49' does not match format '%Y-%m-%d %H:%M:%S +0000'
-   same for `$ /usr/bin/gstlal_injsplitter --nsplit 1 --usertag BNS --output-path filter/split_injections bns_injections.xml`
-   not when prefixing `singularity exec -B $TMPDIR,$PWD $GSTLAL_IMG`
-   CONCLUSION: different version of GstLAL - command was not executed in singularity


<a id="org5d5cb1c"></a>

## Diagnose the issue

-   Run some simple copy commands and bash scripts to probe the environment on the compute nodes
-   do (roughly) the same in the bash scripts as in the bare commands (2x)
-   with and without singularity (2x)
-   total: 4 incarnations
-   NOTE: specify paths everywhere, e.g. `/usr/bin/cp`
-   RESULTS:
    1.  no error when running outside a Singularity container
    2.  errors when (trying to) run(ning) inside, e.g.
        
            Error from slot1_4@wn-lot-062.nikhef.nl: STARTER at 145.107.5.62 failed to send file(s) to
            <145.107.7.239:9618>: error reading from /var/lib/condor/execute/dir_7370/file2.out: (errno 2)
            No such file or directory; SHADOW failed to receive file(s) from <145.107.5.62:36330>
    3.  jobs are executed in the Condor scratch dir in `/pbs/condor/execute/dir_X`, where X is some 4-5-digit
        number.
        -   the input files are transferred there before execution, and the output files are transferred from
            there afterwards
    4.  however, in a Singularity container, the job is run in the user's **HOME dir**
        -   ISSUE: because the input files are not found there, the run cannot succeed (when I/O is involved)
        -   SOLUTION: set `SINGULARITY_TARGET_DIR = /srv` in the HTCondor singularity config on the compute
            nodes
        -   this adds `--pwd /srv` to the `singularity exec ...` call, causing the code to run in the Condor
            scratch dir, where the I/O files are
        -   `/srv` is a **magical link** to the Condor scratch dir(!) and the ill-documented key to the solution
            of this issue


<a id="orgacaba89"></a>

# Use correct data server

-   First two jobs now run in a container for some seconds before crashing
-   `split_injections` (job1) succeeds but `gstlal_reference_psd` (job2) fails
    
            Traceback (most recent call last):
            <SNAP>
          File "/usr/lib64/python3.6/http/client.py", line 974, in send
            self.connect()
          File "/usr/lib64/python3.6/http/client.py", line 946, in connect
            (self.host,self.port), self.timeout, self.source_address)
          File "/usr/lib64/python3.6/socket.py", line 704, in create_connection
            for res in getaddrinfo(host, port, 0, SOCK_STREAM):
          File "/usr/lib64/python3.6/socket.py", line 745, in getaddrinfo
            for res in _socket.getaddrinfo(host, port, family, type, proto, flags):
        socket.gaierror: [Errno -2] Name or service not known
    
    -   ISSUE: using `data-find-server: ldr.ldas.cit:80` in `config.yml`
        -   ping ldr.ldas.cit doesn't work
    -   SOLUTION: replace with `data-find-server: datafind.ligo.org:443`
        -   `ping datafind.ligo.org` works


<a id="orgfbb1c2e"></a>

# Ensure data directory is mounted and bound

-   `gstlal_reference_psd` (job2) still fails:

    **
    ERROR:framecpp_channeldemux.cc:819:gboolean sink_event(GstPad*, GstObject*, GstEvent*): assertion failed: (GST_ELEMENT(element)->numsrcpads > 0)

-   Ron Tapia, Patrick Godwin:
    -   no access to Frame files?
    -   run test script
    -   MvdS: `gstlal_reference_psd_test.sh` with command taken from `full_inspiral_dag.sh`:
        
            /usr/bin/gstlal_reference_psd -vvv --gps-start-time 1187000000 --gps-end-time 1187001000 --channel-name H1=GWOSC-16KHZ_R1_STRAIN --channel-name L1=GWOSC-16KHZ_R1_STRAIN --data-source frames --psd-fft-length 8 --frame-segments-name datasegments --frame-type H1=H1_GWOSC_O2_16KHZ_R1 --frame-type L1=L1_GWOSC_O2_16KHZ_R1 --data-find-server datafind.ligo.org:443 --frame-segments-file segments.xml.gz --write-psd H1L1-GSTLAL_REFERENCE_PSD-1187000000-1000.xml.gz
            
            **
            ERROR:framecpp_channeldemux.cc:819:gboolean sink_event(GstPad*, GstObject*, GstEvent*): assertion failed: (GST_ELEMENT(element)->numsrcpads > 0)
            /srv//condor_exec.exe: line 3:    15 Aborted                 /usr/bin/gstlal_reference_psd -vvv --gps-start-time 1187000000 --gps-end-time 1187001000 --channel-name H1=GWOSC-16KHZ_R1_STRAIN --channel-name L1=GWOSC-16KHZ_R1_STRAIN --data-source frames --psd-fft-length 8 --frame-segments-name datasegments --frame-type H1=H1_GWOSC_O2_16KHZ_R1 --frame-type L1=L1_GWOSC_O2_16KHZ_R1 --data-find-server datafind.ligo.org:443 --frame-segments-file segments.xml.gz --write-psd H1L1-GSTLAL_REFERENCE_PSD-1187000000-1000.xml.gz
    -   The file `tmpj62n7hwo.cache` is transferred back and contains
        
            H H1_GWOSC_O2_16KHZ_R1 1186996224 4096 file://localhost/cvmfs/gwosc.osgstorage.org/gwdata/O2/strain.16k/frame.v1/H1/1186988032/H-H1_GWOSC_O2_16KHZ_R1-1186996224-4096.gwf
            H H1_GWOSC_O2_16KHZ_R1 1187000320 4096 file://localhost/cvmfs/gwosc.osgstorage.org/gwdata/O2/strain.16k/frame.v1/H1/1186988032/H-H1_GWOSC_O2_16KHZ_R1-1187000320-4096.gwf
            L L1_GWOSC_O2_16KHZ_R1 1186996224 4096 file://localhost/cvmfs/gwosc.osgstorage.org/gwdata/O2/strain.16k/frame.v1/L1/1186988032/L-L1_GWOSC_O2_16KHZ_R1-1186996224-4096.gwf
            L L1_GWOSC_O2_16KHZ_R1 1187000320 4096 file://localhost/cvmfs/gwosc.osgstorage.org/gwdata/O2/strain.16k/frame.v1/L1/1186988032/L-L1_GWOSC_O2_16KHZ_R1-1187000320-4096.gwf
-   PG: try `FrChannels /cvmfs/gwosc.osgstorage.org/gwdata/O2/strain.16k/frame.v1/H1/1186988032/H-H1_GWOSC_O2_16KHZ_R1-1186996224-4096.gwf`
    
        $ FrChannels /cvmfs/gwosc.osgstorage.org/gwdata/O2/strain.16k/frame.v1/H1/1186988032/H-H1_GWOSC_O2_16KHZ_R1-1186996224-4096.gwf
        *** Cannot open file /cvmfs/gwosc.osgstorage.org/gwdata/O2/strain.16k/frame.v1/H1/1186988032/H-H1_GWOSC_O2_16KHZ_R1-1186996224-4096.gwf
        *** FrError: in FrFileIOpen Open file error (No such file or directory) for file /cvmfs/gwosc.osgstorage.org/gwdata/O2/strain.16k/frame.v1/H1/1186988032/H-H1_GWOSC_O2_16KHZ_R1-1186996224-4096.gwf
        
        $ lls /cvmfs/gwosc.osgstorage.org
        ls: cannot access /cvmfs/gwosc.osgstorage.org: No such file or directory
-   SOLUTION: `/cvmfs/gwosc.osgstorage.org` must be mounted
    -   NOTE: it must also be added to the Singularity bind list
    -   other urls?


<a id="org5fd2734"></a>

# Upgrade HTCondor

-   `gstlal_median_of_psds` (job3) fails.  `gstlal_median_of_psds_00000-1099-0.err`:

    Traceback (most recent call last):
      File "/usr/bin/gstlal_median_of_psds", line 38, in <module>
        for ifo, psd in read_psd(f, verbose=options.verbose).items():
      File "/usr/lib64/python3.6/site-packages/gstlal/psd.py", line 239, in read_psd
        contenthandler=lal.series.PSDContentHandler
      File "/usr/lib64/python3.6/site-packages/ligo/lw/utils/__init__.py", line 427, in load_filename
        with open(filename, "rb") as fileobj:
    FileNotFoundError: [Errno 2] No such file or directory: 'reference_psd/11870/H1L1-GSTLAL_REFERENCE_PSD-1187000000-1000.xml.gz'

-   ISSUE: `preserve_relative_paths`: files in subdirs don't work properly
-   SOLUTION: upgrade HTCondor to v8.9(.13) (from our v8.8)
    -   this also adds the `ON_EXIT` flag for `WhenToTransferOutput`

