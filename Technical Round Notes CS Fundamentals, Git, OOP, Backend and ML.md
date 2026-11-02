# Technical Round Notes: CS Fundamentals, Git, OOP, Backend and ML

These notes cover the technical-round (TR) material from your master checklist: Sections 2, 3, 6 (theory and patterns), 7 and 8. Each topic is written the way you would answer aloud: a one-line definition, how it works, an example, the trade-off, and the follow-up questions interviewers tend to ask next. Where something connects to a project you built, I have marked it so you can reach for it as evidence.

## How to answer in a technical round

- Start with a one-sentence definition. Then give the mechanism, then a concrete example, then the trade-off.
- Say 'it depends' only when you follow it with what it depends on.
- If you do not know, say what you do know around it and reason out loud. Bluffing is the fastest way to lose a round.
- Expect the chain: definition, why does it exist, what breaks without it, how would you implement it, how did you use it. Prepare each topic to that depth.
- Tie a theory answer to something you built whenever it is honest to do so. 'In my appointment system I prevented double booking with an atomic compare-and-swap update' is stronger than any textbook sentence.

## 1. Operating Systems

### OS basics and system calls

- The OS manages hardware resources (CPU, memory, storage, I/O) and gives programs a clean interface to them. Core jobs: process management, memory management, file systems, device I/O, protection and security.
- **Kernel mode vs user mode:** privileged instructions (touching hardware, page tables) run in kernel mode only. A user program asks the kernel for service through a **system call**, which triggers a trap and switches the CPU to kernel mode, runs the handler, then returns. Examples: `fork`, `exec`, `wait`, `open`, `read`, `write`, `mmap`, `socket`.
- **Interrupts** are signals from hardware (or software traps) that make the CPU stop, save state and run a handler. System calls, page faults and timer ticks all use this mechanism.
- Monolithic kernel (Linux: everything in kernel space, fast, larger crash surface) versus microkernel (minimal kernel, services in user space, safer, more IPC overhead).
- `fork()` creates a child as a copy of the parent (using copy-on-write pages, so it is cheap). `exec()` replaces the process image with a new program. A shell runs commands as fork then exec. A **zombie** is a finished child whose parent has not called `wait`. An **orphan** is a child whose parent died, adopted by init.

### Process vs thread

|  | Process | Thread |
| --- | --- | --- |
| Address space | Own, isolated | Shared with the other threads in the process |
| Creation and switch cost | Heavy | Light |
| Communication | IPC (pipes, sockets, shared memory, message queues, signals) | Shared memory directly, needs synchronisation |
| Failure | Isolated, one crash rarely kills others | A bad thread can crash the whole process |

- A thread has its own stack, registers and program counter. Heap, code, globals and open files are shared.
- The **PCB** (process control block) stores process state, PID, program counter, registers, scheduling info, memory maps and open files.
- States: new, ready, running, waiting (blocked), terminated.
- User-level threads are managed by a library (cheap switches, but one blocking call can block all). Kernel-level threads are managed by the OS (true parallelism on multiple cores).
- **Python note:** the GIL lets only one thread execute Python bytecode at a time, so threads help for I/O-bound work but not CPU-bound work. For CPU-bound work use `multiprocessing` or push the work to separate worker processes. This is the reason you moved heavy work out of Flask into Celery workers in your projects, and you can say so.

### Context switching

- Saving the state of the running process or thread (registers, program counter, stack pointer) into its PCB and loading another's. It is pure overhead, since no useful work happens during it.
- Process switches are costlier than thread switches because they also change the address space (TLB flush, cache effects). Triggered by timer interrupts, blocking I/O, or higher-priority arrivals.

### CPU scheduling

- **Goals and metrics:** CPU utilisation, throughput, turnaround time (completion minus arrival), waiting time (turnaround minus burst), response time (first run minus arrival).
- **FCFS:** simple, non-preemptive, suffers from the convoy effect (short jobs stuck behind a long one).
- **SJF:** minimum average waiting time among non-preemptive schedulers, but needs burst-time prediction and can starve long jobs. **SRTF** is the preemptive version.
- **Priority:** can starve low priority, fixed with **aging** (raise priority with waiting time).
- **Round Robin:** each process gets a time quantum, good response time, preemptive. A huge quantum degrades to FCFS, a tiny one wastes time on switches.
- **Multilevel feedback queue:** several queues, jobs move between them based on behaviour. Real systems (including Linux's CFS, which tracks fair virtual runtime) are variations on this.
- Preemptive means the OS can take the CPU away. Non-preemptive means a process keeps it until it blocks or finishes.
- Be ready for a small numeric: given arrival and burst times, draw the Gantt chart and compute average waiting time. Practise one each for FCFS, SJF, SRTF and RR.

### Concurrency, synchronisation, critical sections

- A **critical section** is code that touches shared state. A correct solution needs mutual exclusion, progress (someone can enter if nobody is inside) and bounded waiting (no starvation).
- **Mutex vs semaphore:** a mutex is a lock with ownership, only the thread that locked it should unlock it, used for mutual exclusion. A semaphore is a counter with `wait` (decrement, block at zero) and `signal` (increment). A binary semaphore looks like a mutex but has no ownership. Counting semaphores guard a pool of N resources and are also used for signalling between threads.
- **Monitor:** a language-level construct that bundles shared data, the procedures that use it, and a lock, with condition variables (`wait`, `notify`) inside. Java's `synchronized` plus `wait/notify` is one. Always wait inside a `while` loop, not `if`, because of spurious wakeups.
- **Spinlock vs blocking lock:** a spinlock busy-waits (good for very short critical sections on multicore), a mutex puts the thread to sleep (good when the wait is long).
- **Classic problems:** producer-consumer (bounded buffer with `empty`, `full` semaphores and a mutex), readers-writers (readers share, writers exclusive, watch for writer starvation), dining philosophers (break the circular wait by ordering the forks, or allow only N-1 philosophers at the table).

```text
producer: wait(empty); wait(mutex); add item; signal(mutex); signal(full)
consumer: wait(full);  wait(mutex); remove item; signal(mutex); signal(empty)
```

### Race conditions, thread safety, CAS, TOCTOU

- A **race condition** is when the result depends on the unlucky timing of threads or processes. The textbook case: two threads run `counter++`, which is really read, add, write. Both read 5, both write 6, one update is lost.
- **Thread safe** code behaves correctly when called from several threads at once. Ways to get there: avoid shared state, immutability, thread-local storage, locks, atomics, or confining state to one thread (message passing).
- **Compare-and-swap (CAS):** one atomic hardware instruction, `CAS(addr, expected, new)`: set to `new` only if the current value equals `expected`, and report success or failure. It is the basis for lock-free structures and atomic counters. The usual pattern is a retry loop: read, compute, CAS, retry on failure.
- **ABA problem:** a value changes A to B and back to A, so CAS succeeds even though state changed in between. Fixed with a version tag or counter next to the value.
- **Optimistic vs pessimistic concurrency:** pessimistic takes locks up front. Optimistic assumes conflicts are rare, does the work, then validates at commit (CAS, or a `version` column in a database). Optimistic wins under low contention, pessimistic under high contention.
- **TOCTOU (time-of-check to time-of-use):** you check a condition and then act on it, but the condition can change in between. Classic examples: check that a file exists then open it (an attacker swaps in a symlink), check that a seat is free then book it (two users both pass the check).
- The fix is to make check and act a single atomic step: an atomic `UPDATE ... WHERE status = 'free'` that you verify by row count, a unique constraint, a transaction with the right isolation level, or `open` with flags instead of a separate existence check.
- **Project evidence:** your appointment system books a slot with `UPDATE slots SET status='held' WHERE id=:id AND status='free'` and checks how many rows were affected. That is a lock-free CAS state machine that removes the TOCTOU window. Your TaskFlow `version` column is the same idea at the row level (optimistic locking against lost updates). Both are strong answers to 'how do you prevent double booking or lost updates'.

### Deadlocks

- A set of processes each holding a resource while waiting for one held by another, so none can proceed.
- **Four necessary (Coffman) conditions, all at once:** mutual exclusion, hold and wait, no preemption, circular wait.
- **Prevention:** break one condition. Most practical is breaking circular wait by imposing a global lock order. Others: request everything at once (breaks hold and wait), allow preemption, use `tryLock` with timeout.
- **Avoidance:** the system only grants a request if the state stays safe. **Banker's algorithm** checks whether some order exists in which every process can finish given its maximum claim. Know how to compute the safe sequence from Allocation, Max and Available matrices.
- **Detection and recovery:** build a resource allocation graph or wait-for graph and look for a cycle. Recover by killing processes or preempting resources. Many systems (and the ostrich approach) simply ignore it and rely on timeouts.
- **Livelock:** threads keep reacting to each other and make no progress, although they are not blocked. **Starvation:** one thread never gets the resource.

### Memory management, paging, virtual memory

- **Contiguous allocation** suffers external fragmentation (free space split into unusable holes). **Paging** splits logical memory into fixed-size pages and physical memory into frames, removing external fragmentation but leaving some **internal fragmentation** in the last page. **Segmentation** splits by logical unit (code, stack, heap) and has external fragmentation.
- **Address translation:** the logical address is split into page number and offset. The page table maps page number to frame number. Physical address is frame number plus offset.
- **TLB:** a small hardware cache of recent translations. A TLB hit avoids the memory access to the page table. Effective access time = hit ratio times TLB plus memory time, plus miss ratio times (extra page-table lookup). Know this formula for numeric questions.
- **Multi-level page tables** keep the table small for sparse address spaces. **Inverted page tables** have one entry per frame.
- **Virtual memory:** gives each process a large private address space, with only the in-use pages kept in RAM. Implemented through **demand paging**: a page is loaded only when touched.
- **Page fault steps:** the MMU finds the page not present, traps to the OS, the OS checks the access is valid, finds a free frame (or evicts a victim), reads the page from disk, updates the page table, and restarts the instruction.
- **Page replacement:** FIFO (simple, suffers Belady's anomaly, where more frames can mean more faults), Optimal (evict the page used farthest in future, the benchmark, not implementable), LRU (evict the least recently used, good in practice, needs hardware support or approximation), Clock or second chance (cheap LRU approximation using a reference bit). Practise counting faults for a reference string with 3 frames.
- **Thrashing:** the system spends most of its time swapping pages because the combined working sets exceed RAM. CPU utilisation drops, which can tempt the scheduler to add more processes and make it worse. Fixes: working-set model, page-fault-frequency control, fewer processes, more RAM.
- **Other useful items:** copy-on-write (parent and child share pages until one writes), memory-mapped files (`mmap`), stack (function frames, auto-managed) versus heap (dynamic, manual or garbage-collected), memory leaks, dangling pointers, and swap space.

## 2. DBMS

### Basics, keys and constraints

- A DBMS stores, retrieves and protects data with concurrency control, recovery, security and a query language. Compared with plain files: less redundancy, integrity constraints, concurrent access, crash recovery, and declarative queries.
- **Three-schema architecture:** physical (how data is stored), logical (tables and relations), view (what each user sees). Data independence means changing one level without breaking the one above.
- **Keys:** super key (any set of columns that identifies a row), candidate key (a minimal super key), primary key (the chosen candidate, unique and not null), alternate key (the other candidates), foreign key (references a key in another table), composite key (several columns together), surrogate key (an artificial id with no business meaning).
- **Constraints:** `NOT NULL`, `UNIQUE`, `PRIMARY KEY`, `FOREIGN KEY` (with `ON DELETE CASCADE`, `SET NULL` or `RESTRICT`), `CHECK`, `DEFAULT`. Put as many rules as you can in the database, because application checks alone can be bypassed and are exposed to races.

### Normalisation and functional dependencies

- A **functional dependency** X to Y means that the same X always determines the same Y. The **closure** of an attribute set is everything it can determine, and you use it to find candidate keys.
- **Anomalies** without normalisation: insertion (cannot record a fact without an unrelated one), update (change in many places, risk of inconsistency), deletion (removing one fact destroys another).
- **1NF:** atomic values, no repeating groups. **2NF:** 1NF plus no partial dependency (no non-key attribute depends on part of a composite key). **3NF:** 2NF plus no transitive dependency (non-key depends only on the key). **BCNF:** for every dependency X to Y, X is a super key. Quick memory line: every non-key attribute depends on the key, the whole key, and nothing but the key.
- A good decomposition is **lossless** (joining gives back the original) and ideally **dependency preserving**. BCNF may lose dependency preservation, 3NF always has a lossless, dependency-preserving decomposition.
- **Denormalisation** is a deliberate step back for read performance (precomputed totals, duplicated columns). Cost: write complexity and risk of drift. Your TaskFlow `weekly_reviews` table, filled by a Celery Beat ETL job for reads, is a deliberate, controlled denormalisation, so use it as the example.

### ER diagrams

- Entities (rectangles), attributes (ovals, with key attributes underlined), relationships (diamonds), cardinality (1:1, 1:N, M:N), participation (total or partial), weak entities (no key of their own, depend on an owner).
- Mapping to tables: each entity becomes a table. 1:N puts the foreign key on the N side. M:N needs a junction table holding both foreign keys (plus relationship attributes). A weak entity's key is the owner's key plus its partial key.
- Practise drawing and converting a small domain (library, hospital, e-commerce) in five minutes.

### Transactions and ACID

- A **transaction** is a logical unit of work, either all of it happens or none of it.
- **Atomicity:** all or nothing (implemented with undo logging or write-ahead logs). **Consistency:** the database moves from one valid state to another, respecting constraints. **Isolation:** concurrent transactions behave as if they ran in some serial order. **Durability:** once committed, it survives crashes (write-ahead log flushed to disk).
- Transfer example: debit A and credit B inside one transaction. If the credit fails, the debit is rolled back.
- Transaction states: active, partially committed, committed, failed, aborted. SQL control: `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT`.
- **Recovery:** write-ahead logging (log the change before applying it), checkpoints, and on restart redo committed work and undo uncommitted work (ARIES style).

### Isolation levels and concurrency anomalies

- **Dirty read:** reading another transaction's uncommitted change. **Non-repeatable read:** reading the same row twice and getting different values because someone committed in between. **Phantom read:** re-running a range query and seeing new rows. **Lost update:** two read-modify-write cycles overwrite each other.

| Level | Dirty read | Non-repeatable read | Phantom |
| --- | --- | --- | --- |
| Read Uncommitted | possible | possible | possible |
| Read Committed | prevented | possible | possible |
| Repeatable Read | prevented | prevented | possible in the standard (prevented in MySQL InnoDB and PostgreSQL snapshots) |
| Serializable | prevented | prevented | prevented |

- PostgreSQL defaults to Read Committed, MySQL InnoDB to Repeatable Read. Higher isolation costs throughput, so choose the weakest level that is correct for the operation.
- **Lost-update defences:** atomic updates (`SET qty = qty - 1`), `SELECT ... FOR UPDATE` row locks, optimistic locking with a `version` column (`UPDATE ... WHERE id = ? AND version = ?`, retry if zero rows changed), or serializable isolation with retry.

### Serializability, locking, MVCC

- A schedule is **serializable** if its effect equals some serial schedule. **Conflict serializable** means you can reach a serial schedule by swapping non-conflicting operations. Test with a **precedence graph**: draw an edge Ti to Tj for each conflicting pair (read-write, write-read, write-write) where Ti goes first. No cycle means conflict serializable. **View serializability** is more general but harder to test.
- **Two-phase locking (2PL):** a growing phase (acquire locks) then a shrinking phase (release). Guarantees conflict serializability but can deadlock. **Strict 2PL** holds write locks until commit, which also avoids cascading aborts.
- **Timestamp ordering:** each transaction gets a timestamp and conflicting operations must respect that order, otherwise someone aborts.
- **MVCC (multi-version concurrency control):** readers see a consistent snapshot while writers create new row versions, so readers do not block writers. This is how PostgreSQL and InnoDB work. The price is vacuum or purge work for old versions.
- **Database deadlock:** two transactions each lock a row the other wants. Detected with a wait-for graph, one victim is aborted and should be retried. Reduce them by locking rows in a consistent order, keeping transactions short, and indexing so fewer rows get locked.

### Indexing

- An index is a separate sorted structure that lets the database find rows without scanning the table. Reads get faster, writes and storage get more expensive.
- **B+ tree** is the default: balanced, wide nodes (few disk reads), all values in sorted leaves linked together, so it supports equality, range scans, `ORDER BY` and prefix matches. Height is typically 3 to 4 even for millions of rows. **Hash index:** equality only, no ranges.
- **Clustered:** the table rows are stored in the index order (one per table). **Non-clustered:** the leaf holds a pointer or the primary key to find the row. A **covering index** contains every column the query needs, so the table is never touched.
- **Composite index and leftmost prefix:** an index on `(user_id, status, due_at)` serves queries filtering on `user_id`, on `user_id, status`, and on all three, but not on `status` alone. Put equality columns first, then the range column.
- **Selectivity:** an index on a column with few distinct values (a boolean) helps little. Do not index everything, since each index slows every insert and update.
- Common reasons an index is ignored: a function on the column, a leading wildcard (`LIKE '%abc'`), an implicit type cast, or low selectivity so a full scan is cheaper.
- **Project evidence:** TaskFlow uses B-tree indexes on `user_id`, `status`, `due_at` and `joinedload` to avoid N+1 queries. Be ready to explain why those columns, which queries they serve, and how you would confirm with `EXPLAIN`.

### Query processing and optimisation

- Pipeline: parse and validate, build a logical plan, the **cost-based optimiser** picks a physical plan using table statistics, then the executor runs it.
- Optimiser ideas: push filters down before joins, choose join order, choose access paths (index scan versus full scan), use statistics for row-count estimates.
- **Join algorithms:** nested loop (fine for small inputs or an indexed inner side), hash join (build a hash table on the smaller side, good for large unsorted equi-joins), merge join (both inputs sorted, good for large sorted inputs).
- Practical tuning: read `EXPLAIN ANALYZE`, add or fix indexes, select only the columns you need, avoid N+1 query patterns, batch writes, use keyset pagination, and watch row estimates that are wildly wrong (stale statistics, run `ANALYZE`).

### SQL vs NoSQL

- **SQL (relational):** fixed schema, joins, strong transactions, mature tooling. Best when data is structured with relationships and correctness matters (payments, bookings, inventory).
- **NoSQL families:** key-value (Redis, DynamoDB: fast lookups, caches, sessions), document (MongoDB: flexible nested records), wide-column (Cassandra: huge write volume, time series), graph (Neo4j: relationship-heavy queries).
- Choose NoSQL for flexible schema, very large horizontal scale, or access patterns that fit the model. Many NoSQL systems trade consistency for availability (see CAP in the interview notes). Most real systems use both: PostgreSQL as the source of truth and Redis as a cache.
- Do not say 'NoSQL is faster'. Say which access pattern each one is good at.

## 3. Computer Networks and the Web

### OSI and TCP/IP models

| OSI layer | TCP/IP layer | Examples |
| --- | --- | --- |
| 7 Application, 6 Presentation, 5 Session | Application | HTTP, HTTPS, DNS, SMTP, FTP, SSH, TLS |
| 4 Transport | Transport | TCP, UDP |
| 3 Network | Internet | IP, ICMP, routers |
| 2 Data link, 1 Physical | Link | Ethernet, Wi-Fi, MAC addresses, switches |

- **Encapsulation:** data becomes a segment (transport header), then a packet (IP header), then a frame (link header). Each layer adds its header on the way down and removes it on the way up.
- Devices: hub (layer 1), switch (layer 2, forwards by MAC), router (layer 3, forwards by IP), load balancer (layer 4 or 7), firewall.

### TCP vs UDP

|  | TCP | UDP |
| --- | --- | --- |
| Connection | Connection-oriented | Connectionless |
| Reliability | Acknowledgements, retransmission, ordering | None, best effort |
| Overhead | Higher | Low |
| Use | Web, email, file transfer, databases | DNS, video calls, gaming, streaming, QUIC |

- **Three-way handshake:** SYN, SYN-ACK, ACK. It synchronises sequence numbers and confirms both sides can send and receive. **Teardown:** four steps (FIN, ACK, FIN, ACK). The side that closes first sits in **TIME\_WAIT** to absorb stray packets.
- **Flow control** (receiver window) stops the sender overwhelming the receiver. **Congestion control** (slow start, congestion avoidance, fast retransmit, AIMD) protects the network.
- A SYN flood attack fills the half-open queue, mitigated with SYN cookies.

### IP addressing, subnetting, ARP, DHCP

- IPv4 is 32 bits. CIDR notation `192.168.1.0/24` means 24 network bits and 8 host bits. Usable hosts = 2^(host bits) minus 2 (network and broadcast address). So /24 gives 254 hosts, /26 gives 62.
- Private ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. **NAT** lets many private hosts share one public IP by rewriting addresses and ports. IPv6 is 128 bits and removes the need for NAT.
- Worked subnetting: split `192.168.1.0/24` into 4 subnets, borrow 2 bits, giving /26 networks at `.0`, `.64`, `.128`, `.192`, each with 62 usable hosts.
- **ARP** resolves an IP address to a MAC address on the local network (broadcast request, unicast reply). **DHCP** hands out IP configuration automatically, in four steps remembered as DORA: Discover, Offer, Request, Acknowledge.

### DNS

- DNS translates names to IP addresses. A lookup walks: browser and OS cache, then the recursive resolver, which asks a root server, then the TLD server (`.com`), then the authoritative server for the domain, and caches the answer for its **TTL**.
- Records: A (IPv4), AAAA (IPv6), CNAME (alias), MX (mail), NS (name server), TXT (verification, SPF).
- Usually UDP port 53, with TCP for large responses. Low TTL gives faster failover and more lookups, high TTL the opposite. DNS is also used for load distribution and geo-routing.

### HTTP, HTTPS, TLS

- **HTTP** is a stateless request-response protocol. A request has a method, path, headers and an optional body. A response has a status code, headers and a body.
- **HTTPS** is HTTP over TLS: encryption (nobody can read it), integrity (nobody can alter it) and authentication (the server is who it claims to be, proven by a certificate signed by a trusted CA).
- **TLS handshake in short:** the client says hello with supported cipher suites, the server replies with its certificate, the client validates the certificate chain, both sides agree on keys using an asymmetric key exchange (ECDHE gives forward secrecy), and from then on they use fast symmetric encryption for the data. TLS 1.3 needs one round trip instead of two.
- **Connection establishment cost:** DNS lookup, TCP handshake, TLS handshake, then the first request. Keep-alive, connection pooling and HTTP/2 multiplexing exist to avoid paying this repeatedly.
- **HTTP/1.1 vs 2 vs 3:** 1.1 handles one request at a time per connection (head-of-line blocking, browsers open several connections). 2 multiplexes many streams over one TCP connection with header compression, but TCP packet loss still stalls all streams. 3 runs over QUIC (UDP), with independent streams, faster handshakes and connection migration.

### Status codes and REST

- 200 OK, 201 Created, 204 No Content. 301 permanent and 302 temporary redirect, 304 Not Modified. 400 Bad Request, 401 Unauthorized (really unauthenticated), 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 429 Too Many Requests. 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout.
- **REST principles:** resources identified by URLs (nouns, `/users/42/orders`), standard methods, stateless requests, a uniform interface, cacheability, and layered architecture.
- **Safe** methods do not change state (GET, HEAD). **Idempotent** methods give the same result when repeated (GET, PUT, DELETE). POST is neither, so retries need an idempotency key. PUT replaces the whole resource, PATCH changes part of it.
- Good API habits: consistent naming, pagination, filtering, versioning (`/v1/`), meaningful error bodies, correct status codes.

### Cookies, sessions, JWT

- **Cookie:** small data the server sets and the browser sends back with each request. **Session:** state kept on the server (in memory or Redis), the browser holds only a session id in a cookie. **JWT:** a signed token (header, payload, signature) that carries claims, so the server can verify it without any lookup.
- Sessions: easy to revoke, but need shared storage when you scale to many servers. JWTs: stateless and easy to scale, but hard to revoke before expiry. Mitigate with short-lived access tokens plus refresh tokens, and a deny-list if you must revoke.
- Cookie flags: `HttpOnly` (JavaScript cannot read it, limits XSS theft), `Secure` (HTTPS only), `SameSite` (limits CSRF). If a JWT sits in `localStorage` it is exposed to XSS, so a `HttpOnly` cookie is often safer.
- A JWT is signed, not encrypted, so never put secrets in the payload. Always verify the signature and the `exp` claim, and pin the accepted algorithm.
- **Project evidence:** PPA V2 uses stateless JWT plus role-based access control for data isolation without server-side session state. Be ready to discuss revocation, expiry and where the token is stored.

### What happens when you type a URL into a browser

1. The browser checks its cache and parses the URL (scheme, host, path).
2. DNS resolution turns the host into an IP (browser cache, OS, resolver, root, TLD, authoritative).
3. The browser opens a TCP connection (three-way handshake), usually through a CDN or load balancer edge.
4. For HTTPS, the TLS handshake follows.
5. The browser sends the HTTP request. The load balancer or reverse proxy forwards it to an app server, which may check a cache, query a database, and build a response.
6. The response comes back with status, headers and body. Caching headers decide what is stored.
7. The browser parses the HTML, builds the DOM and CSSOM, fetches scripts, styles and images (more requests, many in parallel), runs JavaScript, lays out and paints the page.

Add depth where the interviewer pulls: ARP and the default gateway for the first hop, NAT at the router, HTTP/2 multiplexing, CDN edge caching, and the critical rendering path.

## 4. Programming and Computer Systems

### C++ essentials (if the company uses C++)

- **Pointer:** a variable holding a memory address, can be null, can be reseated, supports arithmetic. **Reference:** an alias for an existing object, must be initialised, cannot be null, cannot be reseated. Prefer references for parameters you do not own, pointers when 'no object' is a valid state.
- Pass by value copies, by reference avoids the copy, by `const&` avoids the copy and forbids modification. Returning a reference to a local variable is a classic bug (dangling reference).
- **Stack vs heap:** stack memory is automatic and fast, heap memory (`new`, `malloc`) lives until you free it. Leaks, double frees, use-after-free and dangling pointers are the standard failures.
- **RAII:** acquire a resource in a constructor and release it in the destructor, so cleanup happens automatically even when exceptions fly. Lock guards (`std::lock_guard`), file streams and smart pointers are all RAII.
- **Smart pointers:** `unique_ptr` (sole owner, movable, not copyable), `shared_ptr` (reference counted, shared ownership, beware cycles), `weak_ptr` (non-owning observer that breaks cycles). Prefer `make_unique` and `make_shared`.
- **Rule of three, five, zero:** if you define a destructor, copy constructor or copy assignment, you probably need all three, plus move operations for five. Better: own nothing directly and follow the rule of zero.
- **Virtual functions:** dynamic dispatch through a vtable. A base class used polymorphically needs a `virtual` destructor.
- **STL basics:** `vector` (dynamic array, O(1) amortised push\_back), `deque`, `list`, `set` and `map` (balanced tree, O(log n), ordered), `unordered_set` and `unordered_map` (hash, O(1) average), `priority_queue` (max-heap by default), `stack`, `queue`, and algorithms like `sort`, `lower_bound`, `accumulate`, `next_permutation`. Know iterator invalidation: inserting into a `vector` can invalidate iterators.

### Linux and the command line

- Navigation and files: `ls -la`, `cd`, `pwd`, `cp`, `mv`, `rm -r`, `mkdir -p`, `touch`, `cat`, `less`, `head`, `tail -f` (follow a log).
- Search and text: `grep -rn 'pattern' .`, `find . -name '*.py'`, `awk`, `sed`, `sort | uniq -c`, `wc -l`, `cut`, and pipes (`|`) plus redirection (`>`, `>>`, `2>&1`).
- Permissions: `chmod 755 file` (owner rwx, group r-x, others r-x), `chown`. Read `rwxr-xr--` as 754.
- Processes: `ps aux`, `top` or `htop`, `kill -9 PID` (SIGKILL cannot be caught, `kill` sends SIGTERM first), `jobs`, `bg`, `fg`, `nohup`, `&`.
- Resources and network: `df -h`, `du -sh`, `free -m`, `lsof -i :8000` (who holds the port), `netstat` or `ss -tulpn`, `curl -v`, `ping`, `dig`, `ssh`, `scp`.
- Everything is a file. Know stdin, stdout and stderr (file descriptors 0, 1, 2), environment variables, `PATH`, `systemd` or `journalctl` for services, and `cron` for scheduling.

### Compilers and program execution

- **Compilation stages:** preprocessing (macros, includes), lexical analysis (tokens), syntax analysis (parse tree), semantic analysis (types, scopes), intermediate code, optimisation, code generation, assembly, then **linking** (static or dynamic libraries) into an executable. The **loader** puts the program in memory and starts it.
- **Compiler vs interpreter:** a compiler translates ahead of time to machine code, an interpreter executes step by step. Python compiles to bytecode that the VM interprets, Java compiles to bytecode that a JIT compiles to machine code at runtime.
- **Process memory layout:** text (code), data and BSS (globals), heap (grows up), stack (grows down, holds frames with locals and return addresses). Stack overflow comes from runaway recursion.

### Computer architecture basics

- CPU cycle: fetch, decode, execute, write back. **Pipelining** overlaps stages, hazards (data, control, structural) stall it. Branch prediction reduces control hazards.
- **Memory hierarchy:** registers, L1, L2, L3 cache, RAM, disk, with speed falling and size growing at each step. Caches work because of **locality** (temporal: reuse soon, spatial: nearby addresses). This is why iterating a matrix row by row beats column by column.
- Cache mapping: direct, set associative, fully associative. Write policies: write-through and write-back.
- Endianness (little endian stores the low byte first), two's complement for negative numbers, floating point representation and why `0.1 + 0.2 != 0.3`.
- RISC vs CISC, multicore and cache coherence, and Amdahl's law (speedup is limited by the serial fraction).

## 5. Git and Collaborative Development

### Core model

- Git stores snapshots as commits (each with a hash, a parent, an author and a message). A branch is just a movable pointer to a commit. `HEAD` points to where you are now. Three areas: working directory, staging area (index), repository.
- **Basic loop:** `git init` or `git clone <url>`, `git status`, `git add <file>` (or `-p` to stage hunks), `git commit -m 'message'`, `git log --oneline --graph`, `git diff` (unstaged), `git diff --staged`.
- **Branches:** `git branch name`, `git switch name` (or `checkout`), `git switch -c name` to create and switch, `git branch -d name` to delete.
- **Remotes:** `git remote -v`, `git fetch` (download, change nothing local), `git pull` (fetch plus merge, or fetch plus rebase with `--rebase`), `git push -u origin branch`.

### Undoing things safely

| Command | What it does | Safe on shared branches? |
| --- | --- | --- |
| `git restore file` | Discard unstaged changes in a file | Local only |
| `git restore --staged file` | Unstage | Local only |
| `git reset --soft HEAD~1` | Move branch back, keep changes staged | Local only |
| `git reset --mixed HEAD~1` (default) | Move back, keep changes unstaged | Local only |
| `git reset --hard HEAD~1` | Move back, throw changes away | Local only, destructive |
| `git revert <commit>` | Create a new commit that undoes an old one | Yes, history stays intact |
| `git commit --amend` | Rewrite the last commit | Only if not pushed |

- Rule of thumb: **reset rewrites history (private work), revert adds history (shared work)**.
- **`git reflog`** records where `HEAD` has been. After a bad reset or deleted branch, find the old commit hash in the reflog and `git reset --hard <hash>` or `git switch -c rescue <hash>`. Lost commits are rarely lost.
- `git stash` shelves uncommitted work (`stash push -m`, `stash list`, `stash pop`, `stash apply`).

### Merge, rebase, cherry-pick, bisect

- **Merge** joins two histories with a merge commit (or fast-forwards if there is no divergence), preserving true history. **Rebase** replays your commits on top of another branch for a linear history, rewriting those commits. Do not rebase commits other people have already pulled.
- **Interactive rebase** (`git rebase -i HEAD~4`): squash, reword, reorder, drop commits to clean a branch before a pull request.
- **Squash merge** collapses a feature branch into one commit on main. Fast-forward and no-ff merges are the other styles. Teams pick one convention.
- **Cherry-pick** (`git cherry-pick <hash>`) copies one commit to the current branch, for example a hotfix to a release branch.
- **Bisect** finds the commit that introduced a bug with binary search: `git bisect start`, `git bisect bad`, `git bisect good <hash>`, test, mark good or bad, repeat. `git bisect run <script>` automates it.
- **Tags:** `git tag v1.2.0` or annotated `git tag -a v1.2.0 -m 'release'`, then `git push --tags`. Use semantic versioning (major.minor.patch).

### Team workflow

- **Resolving conflicts:** Git marks `<<<<<<<`, `=======`, `>>>>>>>` in the file. Edit to the intended result, remove markers, `git add`, then `git commit` (merge) or `git rebase --continue`. Pull often so conflicts stay small.
- **`.gitignore`** keeps build output, dependencies (`node_modules`, `venv`), `.env` files and secrets out of the repo. If a secret was committed, rotating it matters more than deleting it, because it stays in history.
- **Branching strategies:** trunk-based development (short-lived branches merged to main quickly, feature flags, strong CI), GitHub flow (branch, pull request, review, merge to main, deploy), Gitflow (develop, release and hotfix branches, heavier, suits versioned releases).
- **Pull requests and reviews:** small focused PRs, a clear description, tests, and linked issue. As a reviewer, comment on behaviour and risk, not style that a linter can handle.
- **GitHub Actions CI/CD:** a workflow file in `.github/workflows/` triggers on `push` or `pull_request`, runs jobs on a runner (checkout, set up the language, install dependencies with a cache, run lint and tests, build a Docker image, deploy on main). Secrets live in repository settings, never in the YAML.

```yaml
name: ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install -r requirements.txt
      - run: pytest -q
```

## 6. OOP, SOLID and Design Patterns (theory)

The LLD problems themselves are in the interview notes. This section is the vocabulary and judgement they rely on.

### The four pillars and related ideas

- **Class and object:** a class is a blueprint (state plus behaviour), an object is an instance of it.
- **Encapsulation:** hide internal state, expose a small interface, keep invariants valid (private fields, validation in setters or constructors).
- **Abstraction:** expose what something does, hide how. Interfaces and abstract classes are the tools.
- **Inheritance:** a subclass reuses and specialises a parent ('is-a'). **Polymorphism:** one interface, many behaviours. Compile-time (overloading) versus runtime (overriding, dynamic dispatch).
- **Interface vs abstract class:** an interface is a pure contract (a class can implement many), an abstract class can hold shared state and partial implementation (single inheritance in Java). Pick an interface for capability ('can be paid'), an abstract class for a shared base ('is a payment method with common code').
- **Association** (uses, loose link), **aggregation** (has-a, parts can exist independently, a team and its players), **composition** (has-a, parts die with the owner, a house and its rooms).
- **Composition over inheritance:** deep hierarchies are rigid and fragile. Build behaviour by combining small objects (strategy, decorator), which is also easier to test and change at runtime.
- **Coupling and cohesion:** aim for low coupling (classes know little about each other) and high cohesion (a class does one coherent thing).
- **Dependency injection:** pass dependencies in (constructor) instead of creating them inside. Makes code testable and swappable. Python's FastAPI `Depends` is a ready example from your stack.
- **Immutability:** objects that cannot change after creation are automatically thread safe and easier to reason about (value objects, frozen dataclasses).
- **Error handling:** fail fast with specific exceptions, never swallow errors silently, clean up with `finally` or context managers (`with`), and keep error messages actionable.
- **Thread safety in class design:** document whether a class is thread safe, protect shared mutable state, prefer immutable objects, and avoid leaking `this` during construction. A lazily built singleton needs care.
- **Testing:** unit tests isolate one unit, mocks and fakes replace collaborators, integration tests check the pieces together. Code that takes its dependencies through the constructor is easy to mock, code that calls `datetime.now()` or a global database inside is not.

### SOLID

- **S, Single Responsibility:** a class has one reason to change. A `UserService` that also sends emails and formats reports is three classes pretending to be one.
- **O, Open/Closed:** open for extension, closed for modification. Add a new discount type by adding a class that implements `DiscountStrategy`, not by editing a growing `if/elif` chain.
- **L, Liskov Substitution:** a subclass must be usable wherever the parent is expected without surprising the caller. The classic violation is `Square` extending `Rectangle` and breaking width and height independence. A subclass that throws 'not supported' for an inherited method is a smell.
- **I, Interface Segregation:** many small, specific interfaces beat one fat one. A `Printer` should not be forced to implement `scan()` and `fax()`.
- **D, Dependency Inversion:** high-level modules depend on abstractions, not on concrete low-level details. The `OrderService` depends on a `PaymentGateway` interface, not directly on a Stripe class. Dependency injection is how you wire it.

### Design patterns

| Pattern | Idea | Typical use |
| --- | --- | --- |
| Singleton | One instance with global access. Make creation thread safe. Often overused, a DI-managed single instance is usually better. | Config, logger, connection pool |
| Factory | A function or class decides which concrete type to build, callers use the interface. | Creating payment providers, parsers |
| Builder | Construct a complex object step by step, avoiding constructors with many arguments. | Query builders, request objects |
| Strategy | Swap interchangeable algorithms behind one interface at runtime. | Pricing, sorting, routing, AI provider choice |
| Observer | Subjects notify subscribed observers on change (publish-subscribe in-process). | Event systems, UI updates, notifications |
| Decorator | Wrap an object to add behaviour without changing its class, and wrappers stack. | Logging, caching, auth around a service |
| Adapter | Convert one interface into the one clients expect. | Wrapping a third-party SDK |
| Facade | A simple front for a complex subsystem. | A `checkout()` call that coordinates stock, payment, shipping |
| Command | Turn a request into an object (so it can be queued, logged, undone). | Undo stacks, task queues, job scheduling |
| State | An object changes behaviour as its internal state changes, each state is a class. | Vending machine, order lifecycle, ticket status |
| Repository | A collection-like interface that hides data access from business logic. | Keeping SQL out of services, easy to fake in tests |

- **Project evidence:** TaskFlow uses the Repository pattern and a service layer so routes stay thin, a Circuit Breaker around the ML client, and your appointment system uses a pluggable provider protocol (Strategy) for the AI backend. Each is a legitimate example for 'where have you used a design pattern'.
- For each pattern you should be able to sketch the classes, name the problem it solves, and name one downside.

### UML essentials

- **Class diagram:** boxes with attributes and methods, relationships as lines. Inheritance is a hollow triangle arrow, interface realisation a dashed one, association a plain line, aggregation a hollow diamond, composition a filled diamond. Add multiplicities (1, 0..*, 1..*).
- **Sequence diagram:** participants across the top, time flowing down, arrows for calls and returns. Use it to show one flow, such as `User -> API -> BookingService -> SlotRepository -> DB`.
- Process for any design question: list requirements, find nouns (entities) and verbs (behaviours), assign responsibilities, draw relationships, walk through two or three use cases, then cover edge cases, validation and exceptions.

## 7. Backend Engineering and Deployment

### API design

- Resource-oriented URLs, correct methods and status codes, consistent JSON errors (`{ 'error': { 'code': 'SLOT_TAKEN', 'message': '...' } }`), pagination (`limit` plus cursor), filtering and sorting, versioning, and validation at the boundary (Pydantic in FastAPI, Marshmallow in Flask).
- **Idempotency:** accept an `Idempotency-Key` header on POSTs that create things. Store the key and the result, and return the stored result on a retry, so a client timeout never creates two bookings or two charges. You implemented this in TaskFlow with UUID idempotency keys.
- **Layering:** thin route handlers, a service layer for business rules, a repository layer for data access. It keeps logic testable and the database swappable. Both PPA V2 and TaskFlow follow this, say so.
- **OpenAPI and Swagger:** FastAPI generates the spec and an interactive `/docs` page from type hints. A published spec is documentation, a contract for the frontend, and a source for client generation.

### Authentication and authorisation

- **Authentication** answers 'who are you', **authorisation** answers 'what may you do'. Common models: session cookie, JWT access plus refresh token, OAuth 2.0 (delegated access, for example 'Sign in with Google' or Calendar access), API keys for services.
- **RBAC:** permissions attach to roles, users get roles. Enforce on the server on every request, never only in the UI. Always check object-level ownership too (a user must not read another user's record by changing an id, the classic IDOR bug).
- **Passwords:** hash with bcrypt, scrypt or argon2 (slow, salted). Never store plain text, never use a fast hash like SHA-256 alone.
- **Refresh tokens and stored secrets:** encrypt them at rest. Your appointment system stores Google OAuth refresh tokens with Fernet encryption, which is a direct example.
- Typical attack names and one-line defences: SQL injection (parameterised queries), XSS (escape output, `HttpOnly` cookies, CSP), CSRF (`SameSite`, tokens), brute force (rate limiting and lockout), SSRF (allow-list outbound URLs).

### PostgreSQL schema design and optimisation

- Pick correct types (`timestamptz` for time, `numeric` for money, `uuid` or `bigint` keys), add foreign keys and `NOT NULL`, and put unique constraints where the business needs uniqueness (for example one booking per slot).
- Index the columns you filter and join on, and the composite order that matches your queries. Verify with `EXPLAIN ANALYZE`. Use partial indexes for hot subsets (`WHERE status = 'free'`).
- Fix N+1 queries with joins or eager loading. Use connection pooling (PgBouncer or the ORM's pool). Prisma with Neon in InventoryPro is pooling in practice, so know why serverless needs it: many short-lived function instances can exhaust Postgres connections.
- Use migrations (Alembic, Prisma Migrate) rather than hand-edited schemas, and make migrations backward compatible when deploying without downtime.

### Redis caching and invalidation

- **Cache-aside:** the app checks Redis first, on a miss reads the database, then writes the result to Redis with a TTL. Simple, the cache can fail without breaking correctness. You used this in PPA V2 and TaskFlow.
- Other strategies: write-through (write to cache and database together), write-behind (write to cache, flush later, risk of loss), read-through.
- **Invalidation is the hard part.** Options: TTL expiry, delete the key on write, versioned keys. Delete on write is the usual choice. Know the race where a stale read refills the cache right after the delete.
- **Cache stampede:** a hot key expires and many requests hit the database at once. Defences: a lock so only one request rebuilds (`SETNX` with a TTL, which you used), jittered TTLs, early refresh, serving slightly stale data.
- Also: cache penetration (queries for missing keys, cache a null marker or use a Bloom filter), eviction policies (LRU, LFU), hot keys, and **failing open** when Redis is down (your appointment system degrades gracefully, you can speak to the trade-off).
- Redis is also used for rate limiting, distributed locks, queues and pub-sub, sessions, and as a Celery broker.

### Background jobs and asynchronous processing

- Move slow or unreliable work (emails, report generation, ML inference, third-party calls) out of the request path into a task queue (Celery with Redis or RabbitMQ as the broker). The request returns quickly, a worker does the job.
- **Celery Beat** schedules periodic tasks. In your appointment system it reclaims abandoned slot holds every 60 seconds. In TaskFlow it runs the weekly ETL.
- Design tasks to be **idempotent** (safe to run twice), set retries with exponential backoff and jitter, set time limits, and send failures to a dead-letter store or alert. Delivery is typically at-least-once, which is exactly why idempotency matters.
- **Transactional outbox:** write the business change and an 'event to send' row in the same database transaction, and let a worker publish the events. It avoids the dual-write problem (database updated but message lost, or the reverse). You used this for email delivery.

### Docker and Docker Compose

- An **image** is a read-only layered template, a **container** is a running instance. Containers share the host kernel (lighter than virtual machines, which each run a full OS).
- Dockerfile habits: small base image, copy dependency files and install before copying source (so the layer cache works), run as a non-root user, one process per container, use a `.dockerignore`, and multi-stage builds to keep the final image small.
- **Compose** describes a multi-container app in one file (web, worker, beat, Postgres, Redis), with networks, volumes for persistent data, environment variables, `depends_on` and health checks. Containers talk to each other by service name.
- Key commands: `docker build`, `docker run -p 8000:8000`, `docker ps`, `docker logs -f`, `docker exec -it <c> sh`, `docker compose up -d --build`, `docker compose down -v`.

### CI/CD, testing, configuration

- **CI:** every push runs lint, tests and build automatically. **CD:** passing builds deploy automatically (or with one approval). Keep the pipeline fast, cache dependencies, and block merges when it fails.
- **Testing pyramid:** many fast unit tests, fewer integration tests (real database, in a container), a handful of end-to-end tests. Test the API with `pytest` plus the framework's test client. Test the dangerous concurrency paths explicitly, such as two simultaneous bookings for the same slot where exactly one must succeed.
- **Configuration:** follow the twelve-factor idea. Config lives in environment variables, never in code. Secrets come from the platform's secret store or CI secrets, never the repository. Keep a `.env.example` with names but no values. Have separate settings for development, test and production.
- **Structured logging:** log JSON with a timestamp, level, request id and key fields, so logs can be searched. Log errors with stack traces, never log passwords, tokens or personal data. Propagate a request id across services.
- **Health checks:** a cheap `/health` (process is up, used for liveness) and a `/ready` check that verifies dependencies such as the database and Redis (readiness). Platforms and load balancers use them to restart or stop routing.

### Cloud deployment and troubleshooting

- Know the platforms you used: Railway and Render (managed containers from a Git repo), fly.io (container VMs near users, CLI driven), Vercel (serverless and edge, best for Next.js) with Neon (serverless Postgres).
- Deployment checklist: build passes, environment variables set, database migrated, health check green, logs clean, domain and HTTPS working, background workers actually running (a web process alone does not run Celery), and a rollback plan.
- Troubleshooting order: read the logs, check the health endpoint, check environment variables and secrets, check the database connection and migrations, check memory and CPU limits, check the port binding (`0.0.0.0` and the port the platform expects), then reproduce locally in Docker.
- **Before every interview, open each live link yourself.** A dead demo or a cold-start timeout is a self-inflicted wound. Free tiers sleep, so wake them up shortly beforehand.

## 8. AI/ML and GenAI

### Core machine learning

- **Bias-variance trade-off:** high bias (underfitting, model too simple, poor on train and test), high variance (overfitting, great on train, poor on test). Fix overfitting with more data, regularisation, simpler models, dropout, early stopping, and bagging. Fix underfitting with richer features or a more expressive model.
- **Train, validation, test:** train fits parameters, validation tunes hyperparameters and picks models, test is touched once at the end for an honest estimate. For time series, split by time, never randomly.
- **Data leakage:** information from outside the training data leaks into it. Examples: scaling or imputing on the full dataset before splitting (fit transformers on train only, use a `Pipeline`), features that encode the label, duplicates across splits, future data in a time series. Leakage produces suspiciously good scores that collapse in production.
- **Metrics:** accuracy misleads on imbalanced data. **Precision** = TP / (TP + FP) (of what I flagged, how much was right). **Recall** = TP / (TP + FN) (of what exists, how much I caught). **F1** is their harmonic mean. **Macro F1** averages F1 across classes equally, **micro** pools all predictions, **weighted** weights by class size. **ROC-AUC** is the chance a random positive ranks above a random negative, threshold independent, but optimistic under heavy imbalance, where PR-AUC is more informative. Pick the metric from the cost of errors: fraud and disease want recall, spam filters want precision.
- **Cross-validation:** k-fold (use stratified for classification) gives a more stable estimate than one split. Hyperparameter tuning: grid search, random search (often better for the same budget), Bayesian optimisation, always inside cross-validation.
- **Feature engineering:** scaling (needed for distance and gradient-based models, not for trees), encoding categoricals (one-hot, target encoding with care), handling missing values, text features (TF-IDF, embeddings), interaction and date features.
- **Class imbalance:** class weights, resampling (SMOTE, undersampling, applied only to the training fold), threshold tuning, and choosing PR-based metrics.
- **Regression vs classification:** continuous target (linear regression, evaluate with MAE, RMSE, R^2) versus discrete (logistic regression, trees, evaluate with the metrics above). Logistic regression is a classifier despite the name.
- **Models to be able to explain:** decision trees (split by impurity such as Gini or entropy, easy to overfit), random forest (bagging plus feature randomness, reduces variance), gradient boosting such as XGBoost and LightGBM (sequential trees fitting residuals, strongest on tabular data), SVM (maximum margin, kernels for non-linear boundaries), k-NN, naive Bayes. **Bagging** reduces variance by averaging independent models, **boosting** reduces bias by correcting errors in sequence, **stacking** trains a meta-model on other models' predictions.
- **Gradient descent and regularisation:** update weights against the gradient of the loss, `w = w - lr * grad`. Learning rate too high diverges, too low crawls. Variants: batch, stochastic, mini-batch, Adam. **L2 (ridge)** shrinks weights smoothly, **L1 (lasso)** drives some weights to exactly zero (feature selection).
- **Neural networks:** layers of weighted sums plus non-linear activations (ReLU, sigmoid, softmax). **Backpropagation** applies the chain rule to compute the gradient of the loss with respect to every weight. Know vanishing gradients, dropout, batch normalisation and why non-linearity is essential.
- **Experiment tracking and reproducibility:** fix random seeds, version the data and code, log parameters and metrics (MLflow or Weights and Biases), and save the preprocessing with the model.
- **Project evidence:** your text-classification pipeline (stacking ensemble on 200,000+ records, 0.826 Macro F1), the MCQ solver (frozen RoBERTa embeddings plus BiLSTM attention with LightGBM stacking) and TaskFlow's XGBoost time-estimate model with weekly PSI drift checks (retrain when PSI exceeds 0.20). For each, prepare: why that metric, how you avoided leakage, how you validated, what failed first, and what you would do with more time.

### GenAI and LLM systems

- **Tokenisation:** text is split into subword tokens (BPE, WordPiece). Cost, context limits and some odd model behaviour (spelling, arithmetic) trace back to tokens. **Embeddings:** dense vectors where similar meaning sits close together, compared with cosine similarity or dot product.
- **Attention and transformers:** self-attention lets each token weigh every other token, computed as softmax(QK^T / sqrt(d)) V. Multi-head attention learns several relation types in parallel. Positional encodings give order. Decoder-only models (GPT-style) predict the next token. Cost grows quadratically with sequence length, which is why long context is expensive.
- **Fine-tuning vs RAG vs prompting:** prompting is cheapest and fastest. RAG injects fresh or private knowledge at query time without retraining. Fine-tuning changes behaviour and style (LoRA makes it cheap). Use RAG for knowledge, fine-tuning for behaviour.
- **RAG pipeline:** ingest documents, **chunk** them (size and overlap are a real trade-off: too small loses context, too large dilutes relevance and burns tokens), embed each chunk, store in a vector database, then at query time embed the question, retrieve the top-k chunks, optionally **rerank** with a cross-encoder, and put the best chunks into the prompt with an instruction to answer only from them and to cite. Add metadata filters and hybrid search (BM25 keyword plus vectors) for exact terms such as names and codes.
- **Vector databases and similarity search:** store vectors with metadata and search by nearest neighbour. Exact search is O(n), so production systems use approximate indexes (HNSW graphs, IVF clustering) that trade a little recall for big speed. Examples: FAISS, Chroma, pgvector, Pinecone, Qdrant.
- **Evaluation and hallucination:** a hallucination is fluent output that is not supported by the source or by reality. Reduce it with grounding (RAG with citations), constrained output (schema validation, as in your Marshmallow guardrails in TaskFlow), lower temperature, and refusing when retrieval is weak. Evaluate retrieval (recall at k, MRR) separately from generation (faithfulness, answer relevance, human or LLM-as-judge with spot checks), and keep a fixed test set of questions.
- **Prompt injection and security:** untrusted text (a web page, a document, a resume) can contain instructions that hijack the model. Defences: treat retrieved content as data, separate it from instructions, restrict tools and permissions, validate and sanitise outputs, require confirmation for sensitive actions, never put secrets in prompts, and log for review. Also be aware of data leakage across users in multi-tenant retrieval (filter by tenant in the retrieval layer).
- **Model serving and monitoring:** latency (time to first token and total), throughput, batching, caching of repeated prompts and embeddings, streaming responses, timeouts with a fallback path, and cost per request. Your appointment system's keyword-heuristic fallback when the LLM times out and TaskFlow's circuit breaker around the ML client are good concrete answers. Monitor **data drift** (inputs change, measured with PSI or KS tests) and **concept drift** (the input-output relationship changes), and trigger retraining or alerts when they cross thresholds.
