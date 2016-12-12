# queueing-tool
A tool for scheduling multiple parallelized jobs to a machine. Communication of with the submitted jobs happens via TCP. At submission, all jobs register at a server process which then contacts the jobs if the are allowed to start or need to be stopped/deleted. Jobs are scheduled according to a priority that initially depends on the amount of requested resources and increases with the waiting time of the job.

################################################################################
# (A) INSTALLATION                                                             #
################################################################################

Copy the directory *queue* to any place you like. In principle, this queue can already be used now.
The following tips may simplify the usage.

# (1) Starting the server automatically ########################################

If the server should be started automatically, place the script *qserver_deamon*
in */etc/init.d/* and run
    
    sudo update-rc.d qserver_daemon defaults

The server now is started directly while booting. You can also start/stop it by
invoking */etc/init.d/qserver_deamon start/stop*.
Note that you need to adjust the server arguments in the script for your needs.
The respective block is marked with a *TODO*.

# (2) Global access to the queue-commands  #####################################

The queue-commands are located wherever you placed the *queue* directory.
For direct access, it is recommended to add the following line to your .bashrc:

    export PATH=/<your-queue-path>/queue:$PATH

You might as well put a .sh file containing this line in /etc/profiles.d, so the
path will be loaded for all users.


################################################################################
# (B) USAGE                                                                    #
################################################################################

# (1) qserver ##################################################################

The server manages all submitted jobs and distributes the available resources
among them. On startup, these resources need to be specified with the following
parameters:

    --port PORT             port to listen on (default: 1234)
    --gpus GPUS             comma separated list of available gpu device ids
    --threads THREADS       number of available threads/cores
    --memory MEMORY         available main memory in mb
    --abort_on_time_limit   kill a job if its time limit is exceeded
    
If you use the *qserver_deamon*, you only need to specify the available resources
once in the script you places in */etc/init.d*.


# (2) queue-scripts ############################################################

A queue job is specified by bash scripts that are organized in blocks. A block is a part
of the script that forms an independent job and can be specified as follows:

    #block(name=[jobname], threads=[num-threads], memory=[max-memory], subtasks=[num-subtasks], gpus=[num-gpus], hours=[time-limit])

where
   * [name] is the name of the job (default: unknown-job)
   * [threads] is the number of threads to use (default: 1)
   * [memory] is the maximal amount of memory in mb for the job (default: 1024)
   * [subtasks] is the number of subtasks in the block, see below (default: 1)
   * [gpus] specifies the number of GPUs requested for the job (default: 0)
   * [time-limit] is the maximal runtime of the job in hours (default: 1)

Each block is scheduled [subtasks] times in parallel, which for instance allows
for easy data parallelism. The subtask of the actual job is specified with the
$SUBTASK_ID variable and the total number of subtasks is contained in
$N_SUBTASKS. Subtask IDs range from 1 to $N_SUBTASKS.

If more than one block is specified in one script, each block is considered to
be dependent on it predecessor, i.e. it is not started before all subtasks of
the preceding block have finished. Subtasks within a block can run in parallel.

Example:
--------------------------------------------------------------------------------
    #block(name=block-1, threads=2, memory=2000, subtasks=10, hours=24)
      echo "process subtask $SUBTASK_ID of $N_SUBTASKS"
      ./processData data-dir/data.part-$SUBTASK_ID
      
    #block(name=block-2, threads=10, memory=10000, hours=2)
      ./doSomethingThatRequiresTheResultsFromBlock-1
      ./somethingElse
      echo "processed block-2 after all subtasks of block-1"
--------------------------------------------------------------------------------

The first block has 10 subtasks that all process different data. All of the 10 subjobs
run in parallel if there are enough resources. The second block is not started before
all subtasks of the first block are finished. Note that all block parameters that are not
specified are set to the default values.

When a job starts, a *q.log* directory is created and *stdout* and *stderr* are
redirected into this file.

In order to submit the above script, save it as *your-script-name.sh* and invoke

     qsub your-script-name.sh [param1] [param2] [...]


# (2) job status ###############################################################

* 'r' (running): The requested resources have been allocated and the job runs.
* 'w' (waiting): The requested resources could not yet be allocated and the job waits for execution.
* 'h' (hold):    The job has to wait for other jobs to be finished before it can start.

# (3) queue-commands ###########################################################

** qsub **
Submits the script with optional parameters to the queue. Parameters can be accessed with $1, $2, etc.

    qsub [options...] SCRIPT [param1] [param2] ...

    Options:
    -l, --local
            Execute the script locally
    -b BLOCK, --block BLOCK
            Only submit/execute the specified block
    -s SUBTASK SUBTASK, --subtask SUBTASK SUBTASK
            Arguments for this option are a block name and a
            subtask id.Only submit/execute the subtask id of the
            specified block.
    -f FROM_BLOCK, --from_block FROM_BLOCK
            Submit/execute the specified block and all succeeding blocks
    --server_ip SERVER_IP
        Ip address of the server (default: localhost)
    --server_port SERVER_PORT
        Port of the server (default: 1234)

** qstat [-v] **

Prints all jobs that are currently submitted. If option -v is set, output is verbose, i.e. requested resources per job are also displayed.

** qdel **

Deletes jobs from the queue. A job can only be deleted by its owner or by root. Usage:

    qdel [-h] [--server_ip SERVER_IP] [--server_port SERVER_PORT] [-n | -u] jobs [jobs ...]
    
    Options:
    jobs
        Jobs to be deleted. Is a space separated list of job names, user names, or jod ids.
        For ids (neither -n nor -u is specified), jobs ranges separated by a '-' are also possible.
    --server_ip SERVER_IP
        ip address of the server (default: localhost)
    --server_port SERVER_PORT
        port of the server (default: 1234)
    -n
        delete all jobs of the given names. Asterisks can be used as wildcards.
    -u
        delete all jobs of the given users. Asterisks can be used as wildcards.

