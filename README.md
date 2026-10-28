# Graph path server

A threaded TCP server daemon in C that loads a directed graph and answers path queries between two nodes with breadth-first search, plus a small client. Results are kept in a cache shared by all threads.

Written in 2020 as the final project for CSE 344 System Programming at Gebze Technical University.

## How it works

- Daemon: the server parses its arguments and opens its files while still attached to the terminal, so errors are shown there. It then forks, calls `setsid`, ignores `SIGCHLD` and `SIGHUP`, forks again, sets `umask(0)` and closes every inherited file descriptor except the graph file and the log file.
- Single instance: at startup it creates the named POSIX semaphore `sem-single-instance` with `O_CREAT | O_EXCL`. If it already exists, the server prints `can't create a second instance of the server daemon!` and exits.
- Graph: an edge-list file is loaded into adjacency lists. Lines starting with `#` are comments; every other line is `from<TAB>to`. The node count is the largest id plus one.
- Threads: the main thread accepts connections on 127.0.0.1 and puts each client socket in a queue; a counting semaphore wakes one handler thread from the pool.
- Dynamic pool: the pool starts with `-s` handler threads. A separate resizer thread wakes when 75% of the handlers are busy and grows the pool by 25%, up to `-x`. When every handler is busy the log says so and new connections wait in the queue.
- Cache: a per-source-node linked list of (destination, path string) entries. Only the exact (source, destination) pair is reused, not sub-paths. "No path" answers are cached too.
- Readers and writers: cache lookups and inserts follow the readers-writers pattern with writer priority, built from POSIX semaphores.
- Protocol: the client sends the two node ids as a binary struct of two `unsigned int`s. The server replies with text such as `0->8->1786->146->5171->1338->3179->728->734.` or `path not possible.` and closes the connection.
- Shutdown: on `SIGINT` the server removes the single-instance semaphore, stops accepting, waits for the handler threads, frees everything and exits.
- Log lines and client output carry timestamps.

## Building

```
cd source
make
```

This builds `server` and `client` with `gcc`, `-lpthread` and `-lm`.

## Running

```
./server -i ../data_example.txt -p 34567 -o server.log -s 4 -x 24
./client -a 127.0.0.1 -p 34567 -s 768 -d 979
```

| Server flag | Meaning |
| --- | --- |
| `-i` | graph file |
| `-p` | TCP port; the server listens on 127.0.0.1 only |
| `-o` | log file for all output and errors; an existing file at this path is deleted first |
| `-s` | handler threads at startup |
| `-x` | maximum handler threads (at least `-s`) |

| Client flag | Meaning |
| --- | --- |
| `-a` | server IPv4 address |
| `-p` | server port |
| `-s` | source node |
| `-d` | destination node |

The client prints the path (or `path not possible`) and the response time. Stop the server with `SIGINT`, for example `kill -INT <pid>`.

## Layout

- `source/server.c`: daemon, listener, handler threads, pool resizer, cache locking
- `source/client.c`: client
- `source/graph.h`, `source/queue.h`, `source/cache.h`: adjacency-list graph with BFS, growable circular queue, path cache
- `source/utils.c`, `source/utils.h`: argument parsing and error-checked wrappers
- `data_example.txt`: sample graph
- `report.pdf`, `latex_source.zip`: report and its LaTeX source

## Notes

- `data_example.txt` is the Gnutella peer-to-peer graph `p2p-Gnutella08` (6301 nodes, 20777 edges) from the [Stanford Large Network Dataset Collection](https://snap.stanford.edu/data/).
- The single-instance semaphore is removed only on `SIGINT`. If the server exits any other way, including a bad command line, unlink `sem-single-instance` before starting it again.

## License

MIT, see LICENSE.
