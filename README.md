# server-client-realization

Simple yet powerful server and client realization with synchronization and caching features.

**Context and objective**
I have developed 2 programs. A threaded server and a client. The server loads a graph from a text file, and uses a dynamic pool of threads to handle incoming connections. The client(s) connect to the server and request a path between two arbitrary nodes, and the server provides
this service.

## Server

The server process is a **daemon**, this means it satisfies:

- taking measures against double instantiation (it should not be possible to start 2 instances of the
    server process)
- making sure the server process has no controlling terminal
- closing all of its inherited open files

The server receives the following command line arguments (more to come later):

```
./server -i pathToFile -p PORT -o pathToLogFile
```
**-i**: denotes the relative or absolute path to an input file containing a directed unweighted
graph **such as** the ones from the Stanford Large Network Dataset Collection - Internet peer-to-peer
networks datasets (https://snap.stanford.edu/data/ ) ; an example is included in this archive. The format
is simple, comments start with ‘#’ and the graph is denoted as one edge per row, containing startNode
to destinationNode indices, separated by a tab character.

**-p**: this is the port number the server uses for incoming connections.

**-o**: is the relative or absolute path of the log file to which the server daemon writes all of its output (normal output & errors).

### Main thread

The server process loads the graph to memory from its input file (file
structure is straightforward and explained in it). It is up to you to decide how to model the graph. The server process communicates with the client process(es) through stream/TCP socket based
connections.

The server possess a pool of POSIX threads, and in the form of an endless loop, as soon as a new
connection arrives, it forwards that new connection to an available (i.e. not occupied) thread of the
pool, and immediately continue waiting for a new connection. If no thread is available, then it waits in a blocked status until a thread becomes available.

### Thread pool

The pool of threads are created and initialized at the daemon’s startup, and each thread (executing of course the same function) executes in an endless loop, first waiting for a connection to be handed to them by the server, then handling it, and then waiting again, and so on.


### Handling a connection

Once a connection is established, the client simply sends two indexes i1 and i2, as two non negative integers, representing each one of the nodes of the graph, to the
server, and wait for the server to reply with the path from i1 to i2. Use breadth-first search (BFS) to find a path from i1 to i2 (yes, this approach is suboptimal but mandatory all the same). The indexes of the nodes are those contained in the input file (so don’t fool around naming nodes arbitrarily).

### Efficiency

The graphs that we load from the files, can be potentially very large with millionsof nodes and edges. We made sure our graphs and our BFS implementation are such that requests don’t
take “forever”.

That is why, we avoid recalculating paths if the same request arrives again. If the requested path from i1 to i2 has been already calculated during a past request (by the same or other thread), the thread handling this request should first check a “cache data structure”, containing past calculations, and if the
requested path is present in it, the thread should simply use it to respond to the client, instead of recalculating it.

Assuming we have a path in the cache calculated from 0->3->5->1->2 and now the client requests a path from 3 to 2. Is it to be considered as known or should it be calculated explicitly? It is considered as unknown, and calculated explicitly.

If it is not present in the “cache data structure” then it inevitably calculates it, respond to the client,
and add the newly calculated path into the data structure, so as to accelerate future requests. If no path
is possible between the requested nodes, then the server responds accordingly.

The cache data structure is common to all threads of the pool and can be thought of as a
“database”. It does not containg duplicate entries, and it supports fast access/search. 

This database poses multiple and serious synchronization risks. That is the reason it is implemented in the following readers/writers paradigm, by prioritizing writers. Readers are the threads attempting to search the data
structure to find out whether a certain path is in it or not (will happen often, and will take longer when
the database grows). Writers are the threads that have calculated a certain path and now want to add it
into the data structure (very frequent in the beginning).


### Dynamic Pool

Furthermore, it was mentioned earlier that the pool would be “dynamic”. This
means it will not have a fixed number of threads, instead the number of threads will increase depending
on how busy the server is. Specifically, every time that the load of the server reaches 75%, i.e. 75% of
the threads become busy, then the pool size will be enlarged by 25%. We used an additional thread to keep
track of the pool’s size, server load and pool resizing, we don’t want to occupy the server/main thread
with anything else besides connection waiting.

```
./server -i pathToFile -p PORT -o pathToLogFile -s 4 -x 24
```
-s : this is the number of threads in the pool at startup (at least 2)
-x : this is the maximum allowed number of threads, the pool does not grow beyond this number.

### Output

The server will write all of its output and errors to the log file provided at startup. If it
fails to access it for any reason, we made sure the user is notified through the terminal before the
server detaches from the associated terminal.

```
Example output for the server ( **included a timestamp at the beginning of every line** ):
Executing with parameters:
-i /home/erhan/sysprog/graph.dat
-p 34567
-o /home/erhan/sysprog/logfile
-s 8
-x 24
Loading graph...
Graph loaded in **1.4 seconds** with 123456 nodes and 786889 edges.
A pool of 8 threads has been created
Thread #5: waiting for connection
Thread #4: waiting for connection
Thread #6: waiting for connection
Thread #7: waiting for connection
Thread #3: waiting for connection
Thread #0: waiting for connection
Thread #2: waiting for connection
Thread #1: waiting for connection
A connection has been delegated to thread id #5, system load 12.5%
Thread #5: searching database for a path from node 657 to node 9879
Thread #5: no path in database, calculating 657->
Thread #5: path calculated: 657->786->87->234->
Thread #5: responding to client and adding path to database
...
Thread #4: path not possible from node 5243 to 98
...
Thread #6: path found in database: 445->456->75->6787->
...
System load 75%, pool extended to 10 threads
...
No thread is available! Waiting for one.
...
Termination signal received, waiting for ongoing threads to complete.
All threads have terminated, server shutting down.

If the server receives the SIGINT signal (through the kill command) it waits for its ongoing
threads to complete, return all allocated resources, and then shutdown gracefully.
```


## Client


The client is simple when compared to the server. Given the server’s address and port number, it will
simply request a path from node i1 to i2 and wait for a response, while printing its output on the
terminal.

```
./client -a 127.0.0.1 -p PORT -s 768 -d 979
```
**-a**: IP address of the machine running the server

**-p**: port number at which the server waits for connections

**-s**: source node of the requested path

**-d**: destination node of the requested path

Example output for the client ( **included a timestamp at the beginning of every line** ):

```
Client (4567) connecting to 127.0.0.1:

Client (4567) connected and requesting a path from node 768 to 979
Server’s response to (4567): 768-> 2536 -> 34783 -> 12 ->979, arrived in 0.3seconds.
```

or

```
Client (4569) connected and requesting a path from node 578 to 87234
Server’s response (4569): NO PATH, arrived in 0.19 seconds, shutting down.
```
where 4567, 4569 are the client process PIDs.
