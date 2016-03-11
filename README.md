# qbatch
--------------------------------------------------------------------------------

![Travis CI build status](https://travis-ci.org/pipitone/qbatch.svg?branch=master)

A script for generating array and standard jobs in arbitrary chunks to SGE/PBS clusters.
Can also run jobs locally on systems with no batch capability.

```
usage: qbatch [options] <command_file>

Submits a list of commands to a queueing system. The list of commands can be
broken up into 'chunks' when submitted, so that the commands in each chunk run
in parallel (using GNU parallel). The job script(s) generated by qbatch are
stored in the folder .scripts/

positional arguments:
  command_file          An input file containing a list of commands which will
                        be submitted to the queueing system. A 'command' is
                        simply a call to a command line program or script. Use
                        - to read the command list from STDIN.

optional arguments:
  -h, --help            show this help message and exit
  -t WALLTIME, --walltime WALLTIME
                        Maximum walltime for an array job element or
                        individual job submitted (default: None)
  -c C                  Number of commands from the command list that are
                        wrapped into each job (default: 1)
  -j J                  Number of commands each job runs in parallel. If the
                        chunk size (-c) is smaller than -j then only chunk
                        size commands will run in parallel. This option can
                        also be expressed as a percentage (e.g. 100%) of the
                        total available cores. (default: 8)
  -N JOBNAME, --jobname JOBNAME
                        Override default job name generated from command_file,
                        or set name for STDIN jobs (default: None)
  -w WORKDIR, --workdir WORKDIR
                        Job working directory (default: /projects/jp/qbatch)
  --logdir LOGDIR       Directory to save store log files from batch system or
                        local processes (default: {workdir}/logs)
  -n                    Dry run; nothing is submitted or run (default: False)
  -v                    Verbose output (default: False)

advanced options:
  -o OPTIONS, --options OPTIONS
                        A string containing custom options passed directly to
                        the queuing system. For example, --options "-l vf=8G".
                        This option can be given multiple times. (default: [])
  --header HEADER       A string that will be inserted verbatim at the start
                        of the script. This can be used to insert commands
                        that will be run once per job. This option can be
                        given multiple times. (default: None)
  --afterok_pattern AFTEROK_PATTERN
                        Wait for successful completion of job with name(s)
                        matching glob pattern before starting (default: None)
  --nodes NODES         Nodes to request per job (default: 1)
  --ppn PPN             Number of cores each job requests aka processors per
                        node (ppn on PBS, parallel environment number on SGE).
                        Cores can be over subscribed if -j is larger than
                        --ppn (which is useful when making use of hyper-
                        threading) (default: 8)
  --pe PE               The SGE parallel environment to use if more than one
                        core is reqeuested (via --ppn) for each job. (default:
                        lolparty)
  --highmem             (Scinet-only) Submit to high memory nodes (default:
                        False)
  -i                    Use individual jobs instead of an array job (default:
                        False)
  -b {pbs,sge,local}    The type of queueing system to use. 'pbs' and 'sge'
                        both make calls to qsub to submit jobs. 'local' runs
                        the entire command list (without chunking) directly.
                        (default: pbs)
  --no-env              Do not copy the current environment into the job
                        script(s) (default: False)
```

## Dependencies
qbatch requires at least ``python`` 2.7 and GNU ``parallel`` to run jobs locally.
For Torque/PBS qbatch requires access to ``qsub``, similarly for SGE/SoGE qbatch
requires access to ``qsub``. For Torque/PBS job dependency support the ``pbs_jobnames``
command included in this repository must be found somewhere in the ``$PATH``.


## Environment variable defaults
qbatch supports several environment variables to customize defaults for your
local system

```sh
$ export QBATCH_PPN=12
$ export QBATCH_NODES=1
$ export QBATCH_SYSTEM="pbs"
$ export QBATCH_CORES=$QBATCH_PPN
$ export QBATCH_PE="smp"
```

These correspond to the same named options in the qbatch help output above.


## Some examples:
```sh
# Submit an array job from a list of commands (one per line), default settings
$ qbatch commands.txt
# Generates job files in .scripts/, stores logs in logs/

# Submit an array job for SGE
$ qbatch -b sge commands.txt

# set the walltime 
$ qbatch -w 3:00:00 commands.txt

# Chunk 24 commands per array job
$ qbatch -c24 commands.txt

# Chunk 24 commands per array job, running 12 in parallel
$ qbatch -c25 -j12 commands.txt

# Start jobs after successful completion of all existing jobs with names starting with "2015-11-02-mb_register"
$ qbatch --afterok_pattern '2015-11-02-mb_register*' commands.txt

# Start jobs after successful completion of all existing jobs with names starting with "2015-11-02-mb_register" and "2015-11-02-mb_resample"
$ qbatch --afterok_pattern '2015-11-02-mb_register*' --afterok_pattern '2015-11-02-mb_resample*' commands.txt

# Dynamically generate a list of commands and submit them to batch system
$ for file in /path/to/some/files/*.mnc; echo do_something $file; done | qbatch -N do_something_jobs -
# If the loop runs zero times, qbatch will report a warning to STDERR but exit successfully

# Run jobs locally with GNU Parallel, 12 commands in parallel
$ qbatch -b local -j12 commands.txt
# Many options don't make sense locally: chunking, individual vs array, nodes, ppn, highmem, and afterok_pattern are ignored
```
