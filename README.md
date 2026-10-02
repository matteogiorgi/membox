# Membox

Membox is a concurrent, multi-threaded server written in C (POSIX threads + `AF_UNIX` sockets) that implements an in-memory **object repository**: a store of non-empty byte sequences (*objects*), each identified by a numeric key. Clients connect over a Unix domain socket and can insert, update, read and remove objects, and acquire an exclusive lock on the whole repository.

It was developed by Matteo Giorgi and Andrea Quarta as the final project for the *Laboratorio di Sistemi Operativi * (Operating Systems Lab) course at the Department of Computer Science, University of Pisa; the original specification and the project report are both in Italian:

- [Project specs](membox16.pdf) (`membox16.pdf`)
- [Project report](relazione.pdf) (`relazione.pdf`)




## Features

- **Six operations**: `PUT`, `UPDATE`, `GET`, `REMOVE`, `LOCK`, `UNLOCK`.
- **Fixed thread pool**: a configurable number of worker threads stays alive for the server's whole lifetime. Each client connection is served by a single worker from start to finish.
- **Bounded connection queue**: when every worker is busy, up to `MaxConnections - ThreadsInPool` connections wait in a queue. Any connection beyond that is rejected with `OP_FAIL`.
- **Storage limits**: you can cap the number of stored objects, the total bytes stored and the size of a single object.
- **Readers/writers/lockers synchronization**: many readers can access the repository at the same time, and writer starvation is avoided. A client can also lock the whole repository for itself.
- **Signal handling** through a dedicated thread (no async signal handlers): stop immediately, shut down gracefully, or dump statistics.
- **Statistics** are appended to a file on demand and at shutdown, and a Bash script pretty-prints them.
- **No memory leaks**: the server passes a Valgrind leak check (`make test6`).




## Repository layout

```
.
├── membox16.pdf          project specification (Italian)
├── relazione.pdf         project report (Italian)
├── LICENSE               GPL-3.0
└── src/
    ├── membox.c          server: main/accept loop, worker threads, signal thread
    ├── membox.h          shared state, config parser, reply function, cleanup handler
    ├── read_write.[ch]   readers/writers/lockers library + storage-limit checks
    ├── connections.[ch]  wire protocol: openConnection, readHeader, readData, sendRequest, readReply
    ├── queue.[ch]        integer FIFO on a (resizable) circular array, holds client fds
    ├── errors.[ch]       error-reporting helpers (err_msg, err_exit, err_exit_en, ...)
    ├── icl_hash.[ch]     chained hash table (provided by the course)
    ├── message.h         message header/body types
    ├── ops.h             operation and reply codes
    ├── stats.h           statistics struct and printStats()
    ├── client.c          test client (provided by the course)
    ├── test_hash.c       icl_hash usage example (provided by the course)
    ├── stat.sh           statistics pretty-printer (Bash, no sed/awk)
    ├── test*.sh          test scripts used by the Makefile
    ├── DATA/membox.conf1 config: 6 threads, 32 connections, no storage limits
    ├── DATA/membox.conf2 config: 2 threads, 2 connections, strict storage limits
    └── Makefile
```




## Building

You need `gcc`, `make` and pthreads. `valgrind` is only needed for the leak test. The project was written for Ubuntu x86_64.

```sh
cd src
make            # builds membox, client and test_hash
make cleanall   # removes binaries, objects, libmbox.a, the socket and the stats file
```

The helper modules (`errors`, `connections`, `icl_hash`, `read_write`, `queue`) are compiled into a static library, `libmbox.a`, which is then linked into the server. Compiler flags are `-std=c99 -Wall -pedantic -g`.




## Running

```sh
./membox -f DATA/membox.conf1
```

The server creates the socket at `UnixPath`, prints `SERVER IN ASCOLTO...` ("server listening") and waits for clients.


### Configuration file

Each line has the form `Name = value`. Text after `#` is a comment, and blank lines are skipped. The parser rejects lines with zero or several `=` signs. Names it does not recognize are not errors: they are collected into an "extra" structure that the server could use.

| Option            | Meaning                                                     |
|-------------------|-------------------------------------------------------------|
| `UnixPath`        | path of the `AF_UNIX` listening socket                      |
| `MaxConnections`  | maximum number of concurrent connections (served + queued)  |
| `ThreadsInPool`   | number of worker threads (must be > 0)                      |
| `StorageSize`     | maximum number of objects (`0` = unlimited)                 |
| `StorageByteSize` | maximum total bytes stored (`0` = unlimited)                |
| `MaxObjSize`      | maximum size of a single object (`0` = unlimited)           |
| `StatFileName`    | file the statistics are appended to                         |

`StorageSize` also sets the number of hash-table buckets. When it is `0`, the table gets 1024 buckets.


### Signals

| Signal                          | Effect                                                                                          |
|---------------------------------|-------------------------------------------------------------------------------------------------|
| `SIGINT`, `SIGTERM`, `SIGQUIT`  | stop as soon as possible: workers drop their connections, stats are written, socket is removed  |
| `SIGUSR2`                       | graceful shutdown: stop accepting connections, let the open ones finish, write stats, exit      |
| `SIGUSR1`                       | append the current statistics to `StatFileName` and keep running                                |
| `SIGPIPE`                       | blocked                                                                                         |


### Using the test client

```
./client -l <socket_path> -c key:op:size [-c key:op:size ...] [-s millis]
```

`op` is `0` PUT, `1` UPDATE, `2` GET, `3` REMOVE, `4` LOCK or `5` UNLOCK. `size` is used only by PUT and UPDATE. Operations run in order on a single connection, and the client stops at the first one that fails. `-s` sets a delay between operations. The client exits with `-<reply code>`, so the shell sees `256 - code` (for example, `243` means `OP_PUT_ALREADY`).

```sh
./client -l /tmp/mbox_socket -c 100:0:8192               # PUT 8 KiB under key 100
./client -l /tmp/mbox_socket -c 100:2:0 -c 100:3:0       # GET then REMOVE key 100
./client -l /tmp/mbox_socket -c 0:4:0 -c 7:0:64 -c 0:5:0 # LOCK, PUT, UNLOCK
```

The client fills each object with copies of the key. When it reads an object back with GET, it checks the content and reports a mismatch as an error.




## Protocol

Every field is sent in the host's native binary form (the client and server always run on the same machine).

**Request**

```
header:  op_t op (int)  |  membox_key_t key (unsigned long)
body:    unsigned int len  |  char buf[len]          (PUT and UPDATE only)
```

**Reply**

```
header:  op_t result (int)  |  membox_key_t key (unsigned long)
body:    unsigned int len  |  char buf[len]          (successful GET only)
```

Reply codes (from `ops.h`):

| Code | Name              | Meaning                                          |
|-----:|-------------------|--------------------------------------------------|
| 11   | `OP_OK`           | success                                          |
| 12   | `OP_FAIL`         | generic failure (e.g. too many connections)      |
| 13   | `OP_PUT_ALREADY`  | PUT on a key that already exists                 |
| 14   | `OP_PUT_TOOMANY`  | `StorageSize` reached                            |
| 15   | `OP_PUT_SIZE`     | object larger than `MaxObjSize`                  |
| 16   | `OP_PUT_REPOSIZE` | `StorageByteSize` would be exceeded              |
| 17   | `OP_GET_NONE`     | GET on a missing key                             |
| 18   | `OP_REMOVE_NONE`  | REMOVE on a missing key                          |
| 19   | `OP_UPDATE_SIZE`  | UPDATE with a size different from the stored one |
| 20   | `OP_UPDATE_NONE`  | UPDATE on a missing key                          |
| 21   | `OP_LOCKED`       | repository is locked by another connection       |
| 22   | `OP_LOCK_NONE`    | UNLOCK while the repository is not locked        |

The semantics follow the specification. PUT works only on new keys. UPDATE works only on existing keys and only with the same size. The storage limits block new PUTs but never block the other operations.




## Architecture

### Threads

- **Main thread (the "arbiter")**: parses the configuration, blocks the handled signals, starts the signal thread and the worker pool, and loops on `accept()`. Each accepted descriptor is pushed into the shared connection queue and one sleeping worker is woken. If the queue is full, the arbiter replies `OP_FAIL` and closes the connection itself.
- **Worker threads**: each worker takes a descriptor from the queue, or sleeps on a condition variable while the queue is empty. It then serves that client's requests in a loop until the client closes the connection. On disconnect, the worker releases the repository lock if this client held it, so a client that exits without sending `UNLOCK` cannot leave the repository locked.
- **Signal thread**: waits in `sigwaitinfo()` and handles signals synchronously. To stop the server, it sets a flag (`stop_sigint` or `stop_sigusr2`) and calls `shutdown()` on the listening socket, which makes the arbiter's `accept()` return. The arbiter then broadcasts to the sleeping workers, joins every thread, and runs a cleanup handler (`pthread_cleanup_push/pop`). The handler frees all memory, destroys the repository, writes the final statistics and removes the socket file.

One mutex/condition-variable pair (`lock`, `dormi`) protects the connection queue, the stop flags, the count of busy workers and the statistics.


### Repository access: readers / writers / lockers

All access to the repository goes through one lock, implemented in `read_write.c`. That lock is more than a mutex: it is a readers/writers protocol with a third role, the *locker*.

- `GET` is a **read**. `PUT` is a **write**. `UPDATE` and `REMOVE` first **read** to check that the key exists, then **write**.
- New readers wait while a writer is waiting, so writers do not starve. When a writer finishes it wakes every waiting reader, so readers do not starve either.
- `LOCK` waits until no reader or writer is active, then records which worker owns the repository. Until `UNLOCK`, or until that client disconnects, other workers get `OP_LOCKED`, and the owning worker bypasses the read/write protocol.
- `startWrite()` also enforces the storage limits (`MaxObjSize`, `StorageByteSize`, `StorageSize`, same-size UPDATE). The check runs inside the same critical section that reserves the space, so concurrent PUTs cannot overshoot a limit together.

The repository sits behind a `struct repository` of function pointers (create, find, insert, update, delete, hash, compare, free). Because of this, `icl_hash` could be replaced by another container without changing the worker code. Keys are hashed with 32-bit FNV-1.


### Other modules

- **`queue.c`**: FIFO of integers on a circular array. It keeps head and tail pointers, so it never allocates memory per element. It can optionally double its size when full; the server does not use this.
- **`errors.c`**: `err_msg`, `err_exit`, `err_exit_en` (for pthread error codes) and similar helpers print the `errno` name and description. If the `EF_DUMPCORE` environment variable is set, they call `abort()` to produce a core dump instead of exiting.




## Statistics

On `SIGUSR1` and at shutdown, the server appends one line to `StatFileName`:

```
timestamp - put put_failed update update_failed get get_failed remove remove_failed lock lock_failed connections cur_size max_size cur_objects max_objects
```

`src/stat.sh` displays this file as aligned columns:

```sh
./stat.sh /tmp/mbox_stats.txt            # every column, for every timestamp
./stat.sh /tmp/mbox_stats.txt -p -r      # PUT and REMOVE columns only
./stat.sh /tmp/mbox_stats.txt -m         # max connections, max objects, max size
./stat.sh --help
```

Options: `-p` PUT, `-u` UPDATE, `-g` GET, `-r` REMOVE, `-c` connections, `-s` size, `-o` objects, `-m` maxima. `-m` cannot be combined with the other options. As the specification requires, the script uses neither `sed` nor `awk`.




## Tests

The Makefile has seven test targets, provided by the course. Each one rebuilds everything from scratch and starts its own server:

| Target       | What it checks                                                                          |
|--------------|-----------------------------------------------------------------------------------------|
| `make test1` | basic PUT/UPDATE/GET/REMOVE, 1 MiB objects, duplicate PUT                               |
| `make test2` | concurrent PUTs from two clients: exactly 1000 succeed and 500 fail                     |
| `make test3` | lock protocol: a PUT from another client fails with `OP_LOCKED` while the lock is held  |
| `make test4` | object, byte and count limits, UPDATE size mismatch, `MaxConnections` (`membox.conf2`)  |
| `make test5` | statistics counters                                                                     |
| `make test6` | Valgrind leak check                                                                     |
| `make test7` | stress test: many concurrent clients with mixed operations and frequent `SIGUSR1`       |

The test scripts and `DATA/*.conf` use `/tmp/mbox_socket` and `/tmp/mbox_stats.txt`. If you change these paths, also change `UNIX_PATH` and `STAT_PATH` in the Makefile. Note that `openConnection()` in `connections.c` always connects to the `SOCKET` constant defined in `connections.h` (`/tmp/mbox_socket`) and ignores the path passed with `-l`. If you change `UnixPath`, change that constant as well.




## Authors

- Matteo Giorgi
- Andrea Quarta

The test client, the test scripts, the hash table (`icl_hash`, by Jakub Kurzak) and the skeleton headers were provided by the course instructors (Pelagatti, Torquati).
