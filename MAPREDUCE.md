## The HDFS Map Reduce Interface

Available as module `HDFS.MapReduce`, this map-reduce framework is intended for scenarios where the working dataset would fit into distributed memory (irrespective of the input dataset which can still be arbitarily large). Solving problems where the working dataset would be larger than available memory is still possible, but would require one to implement the sorting and shuffling logic between the map and reduce steps. Using Hadoop map-reduce is a better option in such cases, till such support is available in Julia.


**dmap**( *data_source* , *map_fn* , *collect_fn* ) &rarr; *jobid*

**dmapreduce**( *data_source* , *map_fn* , *collect_fn* , *reduce_fn* ) &rarr; *jobid*

Submits a distributed map or map-reduce job. 
Returns a job id which can be used to reference this job.

**Parameters:**
- data\_source: 
    - represented by an MRInput instance, described in sections below.
- map\_fn:
    - transforms the record read by the reader to output 0 or more records.
- collect\_fn:
    - aggregates the mapped record into a storage area.
- reduce\_fn:
    - reduces multiple collected records to output the final result.
    - reduce\_fn is not required for a pure map operation. The collected results are then left in distributed memory across the nodes and can be referred to by a subsequent run.



**status**( *jobid* ) &rarr; *status_string* 
**status**( *jobid* , *describe* ) &rarr; (*status_string* , *additional_data*)

The *status_string* could be one of:
- *starting*: the job is getting initialized. (may be waiting for the results of another running job)
- *running*: has been scheduled
- *complete*: successfully finished
- *error*: stalled as there was an error while processing

The *additional_info* could be one of the following depending on the current status:
- starting: nothing
- running: percent completion (int)
- complete: nothing
- error: 
    - a string indicating more details
    - an exception if error was set due to an exception
    - any uncaught objects thrown during execution



**results**( *jobid* ) &rarr; ( *status* , *result* )

Returns a tuple of the current status and the reduced result (if present). The result would be set to 'nothing' if there was no reducer of if there was an error.
TODO:
- store(jobid, path): If complete, store the results either in distributed or local form at the given path. Path may point to hdfs:// or file://. Results would be distributed if there was no reduce step.
- load(jobid, path, everywhere=false, reducer=nothing): If stored job present, load stored results to memory. 
    - If results were stored in reduced form, they will be loaded to local node only. Else they are loaded in distributed form on multiple nodes.
    - If results were distributed, all results can be still be loaded at all nodes by specifying everywhere with a reduction function. The distributed results would then be reduced, stored and reduced data sent to all nodes. Henceforth, either load or load\_everywhere may be called without additional reduction step overhead.
- unload(jobid, store/destroy): Unload task from memory. In addition, can optionally be stored to disk or destroyed permanently.



**wait**( *jobid* ) &rarr; *status_code*

Waits for a job to finish. Return status code could be one of:
- 0: error (could be result of an error in the job that was the input to this job)
- 1: starting
- 2: running
- 3: complete 


**times**( *jobid* ) &rarr; ( *total_time* , *wait_time* , *run_time* )

For a completed job, returns the time (seconds) it took:
- total\_time: overall time taken
- wait\_time: time till which the job was not scheduled (possibly waiting for a input job to finish)
- run\_time: time taken after the job was scheduled 



#### Data sources and the reader function
Access to data sources are provided by lower level reader types, specific to each source. All reader types are of the abstract type **MapInputReader**. In the current implementation **HDFSBlockReader** provides functionality to read HDFS files, **FsBlockReader** is used to read disk files and **MapResultReader** assists reading previous map results. 

A data source for map-reduce is an instance of abstract type `MRInput`. Following concrete implementations of it exist:
- MRMapInput(job_list, reader_fn::Function)
    - job_list is a list of job ids pointing to previous map results
    - reader_fn is described below
- MRHdfsFileInput(file_list, reader_fn::Function)
    - file_list is a list of HDFS URLs, pointing to HDFS directories or files. The last part of the URL can optionally contain a regular expression targeted to pick up multiple files.
    - e.g.: `hdfs://username@hdfs\_host:port/folderpath/filepath`, or `hdfs://username@hdfs\_host:port/folderpath`, or `hdfs://username@hdfs\_host:port/folderpath/.*\.csv`
    - reader_fn is described below
- MRFsFileINput(file_list, reader_fn::Function, block_sz::Int)
    - similar to `MRHdfsFileInput` but for regular file system.

The reader\_fn uses a supplied MapInputReader instance to fetch one logical chunk of data. It can optionally include optimimizations to filter only interesting data for the mapper. The reader and the reader\_fn are combined to form an iterator that feeds the map function. The reader function essentially provides all the logic for iterating over records in the file in a single function.

**reader_fn**( *reader* , *state* ) &rarr; ( *record*, *is_done*, *state* )

Where:
- reader: the lower level reader for the data source
- state: the iterator state (set as `nothing` in the first call)

Returns a tuple consisting of:
- record: record read, or `nothing` if no record was found
- is_done: whether there are any more records
- state: iterator state that would be passed back in the next call


#### Map and Collect
Map takes as input the record as provided by the reader function and returns a list of zero or more map results.

**map_fn**( *record* ) &rarr; *[records...]*

Collect is an intermediate step between map and reduce that helps collate results from map into a more compact form. The collect method can also be used as an intermediate reduce step.

**collect_fn**( *result* , *record* ) &rarr; *result*

The result of the map and collect operations are stored in memory by default. If necessary, the collect (and map) operations can use permanent storage and just return references to the location where the objects are stored.



#### Reduce
The results of map and collect are still distributed on every node. The reduce step gathers and combines all these distributed results on to the central node.

**reduce_fn**( *result* , *results...* ) &rarr; *result*

The reduce function takes a final result instance to merge the collected results on to, which may be ‘nothing’ for the very first call to reduce. Reduce may be called multiple times if it is done in phases, distributed across nodes.


### Typical Setup and Execution
- Julia must be setup at identical location on all data nodes.
- Authorized keys setup for ssh from name node machine to all data nodes
- Typically, a job file with all map, collect, and reduce functions is loaded using `require` on the master node, which in turn loads it on all nodes.
  E.g. `require("job_file.jl")`.
  Alternatively, if the functions are simple, anonymous functions can be passed which would get shipped to all nodes.
- Issue one or more mapreduce commands
- Check status and results and issue further commands
- Save results and exit


### Test
A sample test script is provided in the test folder that works on curated twitter data as provided at [infochimps](http://www.infochimps.com/datasets/twitter-census-conversation-metrics-one-year-of-urls-hashtags-sm--2). Comments in the files contain instructions to run the steps.
- twitter\_test\_smileys.jl: Calculates monthly, annual and total counts of smileys. Compares smiley occurrences.
- twitter\_test\_counts.jl: Calculates monthly counts of a search term (regex).


### TODO
- Better scheduling of blocks across processing nodes
- Distributed reduction step
- Scalability improvements, e.g. multiple tasks per node, rack awareness
- Resilience to node failure
- API usability (convenience julia macros/methods to make working with HDFS intuitive)
- Integrate other distributed storage systems: MapR, Cassandra, MongoDB etc


