# System Design Learning Notes

## Scalability
- Clone: 
  - Horizontally scale.
  - Load balancer evenly distributes load.
  - Each server contains the exact same codebase and does not store any user-related data like sessions or profile pictures on local memory.
  - Sessions are stored in an external cache, for example Redis
  - Use Capistrano to deploy all servers at the same time.
  - Create an image file from the servers, for example Amazon Machine Image, as a super clone, then new instances can be based on this for deployment.

- DB:
  1. Master-slave replication: Read from slave, write to master.
    - To scale up: Add more RAM in the master, sharding, denormalization, SQL tuning.
  2. Switch to NoSQL like MongoDB or CouchDB (better option): No joins on the database side.

- Cache:
  - Use in-memory cache like Memcached or Redis. Avoid file-based caching.
  1. Cached Database Queries:
    - Problem: Hard to delete cache for a complex query. When a table cell changes, all the cached queries that may contain that table cell need to be deleted.
  2. Cached Objects (better option):
    - When your class has finished assembling the data array, directly store the data array or the complete instance of the class in the cache.
    - Enables asynchronous processing.

- To be Cached: 
  - User sessions
  - Blog articles
  - Activity streams
  - User-to-friend relationships

- Async:
  1. Use a script to precompute pre-rendered pages hourly and upload them to AWS S3 or another Content Delivery Network.
  2. Frontend sends a task to the queue and informs the user.
     - Have a task queue that workers can pick up.
     - Frontend constantly checks if the task is done.
  - Backends become nearly infinitely scalable, and frontends become fast.
  - If a task is time-consuming, try to do it asynchronously.
  
  - Practice
    - HelloWorld
        - [send.py](HelloWorld/send.py)
        - [receive.py](HelloWorld/receive.py)
        
## Trade-Off
- Performance vs scalability
  - performance problem: slow for single user
  - scalability problem: fast for single user, slow under heavy load.
          ![Alt Text](https://image.slidesharecdn.com/scalabilitypatterns20100510-100512004526-phpapp02/75/scalability-availability-stability-patterns-2-2048.jpg?cb=1665169599)
- Latency vs throughput 
    - Latency: time required to produce some result
    - Throughput: the number of such results produced per unit of time
    - 1 s = 100 * 10^6 clock periods
    - aim for maximal throughput with acceptable latency.

- Availability vs consistency 
    - CAP theorem: choose 2 out of 3
            ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/bgLMI2u.png)
        - Consistency: Every read receives the most recent write or an error
        - Availability: Every request receives a response, without guarantee that it contains the most recent version of the information
        - Partition Tolerance: The system continues to operate despite arbitrary partitioning due to network failures
        
        - AC: centralized system, like RDBMS
        - CP: require atomic read and write, can accept timeout error when waiting for a response from the partitioned node
        - AP: require continue working despite external errors, can accept eventual consistency. write might take some time to propagate when the partition is resolved.
    - Consistency Patterns
        - Weak Consistency: after a write, read may or may not see it. Best effort only. Real time use cases: video chat may lose connection for some time
        - Eventual Consisitency: after a write, read will eventually see it. Data is replicated asyncronously. High availability systems: DNS, email.
        - Strong Consistency: after a write, read will see it. Data is replicated synchornously. Systems need transactions: file systems, RDBMS
        
        ![Alt Text](https://github.com/SitongChe/SystemDesign/blob/main/Consistancy%20Pattern.png?raw=true)
        
   - Availability Patterns
        - Fail-over: 
            - Fail Over Disadvantages
                - more hardware
                - data loss if active system fails before any newly written data replicated to passive
            - Active-passive / master-slave: heartbeats are sent between active and passive server on standby. If the heartbeat is interrupted, the passive server takes over the active's IP address and resumes service
            - Active-active / master-master: both servers are managing traffic, spreading the load between them. DNS/application logic need to know the IP of both servers.
        - Replication
            - master-slave
            - master-master
        - Availability in parallel vs in sequence
            - in sequence: Availability (Total) = Availability (Foo) * Availability (Bar)
            - in parallel(higher): Availability (Total) = 1 - (1 - Availability (Foo)) * (1 - Availability (Bar))
            
    - Scalability Patterns: State
        - Partitioning
        - HTTP Caching
        - RDBMS Sharding
        - NoSQL
        - Distributed Caching
        - Data Grids
        - Concurrency
        
    - Domain Name System:
           ![Alt Text]( https://github.com/donnemartin/system-design-primer/blob/master/images/IOyLj4i.jpg)
        - DNS translate a domain name to an IP address
        - Disadvantages
            - accessing a DNS server introduces a slight delay, although mitigated by caching
            - complex
            - DDos attack
    - Content Delivery Network
        - CDN is globally distributed network of proxy servers, serving content from locations closer to the user. static files, such as HTML, photos, videos are served from CDN. The site's DNS resolution will tell clients which server to contact.
        - Push CDNs: receive new content when change uploaded. User provide content, upload to CDN and rewrite URL to point to the CDN. Content is uploaded only when it is new or changed, minimizing traffic, maximizing storage.
        - Pull CDNs [heavy traffic]: receive new content when user request the content. This result in a slower request until the content is cached on the CDN. A TTL time to live determines how long content is cached. minimize storage, but create redundant traffic if file expired or are pulled before file changed.
            - Sites with heavy traffic should use pull CDNs. As traffic is spread out more evenly with only recently requested content remaining on the CDN.
        - CDN Disadvantages
            - Cost could be high depending on traffic, but if not using CDN, alternative cost as well
            - Content might be stale if updated before TTL expires and the CDN fetches the updated version
            - CDN require changing URL for static content to poin to the CDN
            
    - Load Balancer
         ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/h81n9iK.png)
         - preventing requests from going to unhealthy servers
         - preventing overloading resources
         - helping to eliminate a single point of failure
         - SSL termination. Decrypt incoming requests and encrypt server reponse. remove the need to install X.509 certificates on each server
         - Session persistence. Issue cookies and route a specific client's requests to same instance if the web apps do not keep track of sessions. But it is just basic IP based session, no user profiles.
         - To protect against failures, set up multiple load balancers, with availability patterns of active-passive or active-active
         - hardware(expensive) or software(HAProxy)
         - Layer 4 Load Balancing:
            - transport layer: source, destination IP address, ports.
            - forward network packets to and from the upstream server, performing Network Address Translation
            - require less time and computing source
         - Layer 7 Load Balancing:
            - application layer: contents of the header, message, cookies.
            - terminate network traffic, reads message, makes load balancing decision, open connection to the selected server.
            - Example: direct video traffic to servers hosts video, also direct user billing traffic to security hardened servers
         - Horizontal Scaling Disadvantages:
            - servers should be stateless: they should not contain any user related data like sessions or profile pictures
            - sessons can be stored in a centralized data store such as a data base or a persistent cache(Redis, Memcached)
            - Downstream servers such as caches and databases need to handle more simultaneous connections as upstream servers scale out
        - Load Balancer Disadvantages:
            - load balancer can be a performance bottle neck if it does not have enough resources or not configured properly
            - a single load balancer is a single point of failure, but configuring multiple load balancers further increase complexity
    - Reverse Proxy (web server)
             ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/n41Azff.png)
        - increased security: hide information about backend servers, blacklist IPs, limit number of connections per client
        - increased scalability and flexibility: client only see the reverse proxy's IP, allowing scale servers or change configuration
        - SSL termination: decrypt incoming requests and encrypt server responses so backend servers do not have to perform these expensive operations. remove the need to install X.509 certificates on each server
        - Compression: compress server responses
        - Caching: return the response for cached requests
        - Static content: Serve static conent directly, like HTML photo, videos
        - Load balancer vs reverse proxy:
            - load balancer route traffic to a set of servers serving the same function
            - reverse proxies are useful even for just 1 web/app server
            - NGINX and HAProxy can support both layer 7 reverse proxying and load balancing
        - Reverse Proxy Disadvantages:
            - increased complexity
            - a single reverse proxy is a single point of failure, configuring multiple reverse proxies (failover) further increase complexity
    - Application layer
                 ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/yB5SYwm.png)
                 
        - scale out and configure both web layer and application layer independently.
        - single responsibility principle advocates for small and autonomous services that work together
        - Microservices: a suite of independently deployable, small, modular services. Each service runs a unique process and communicates through a well-defined, lightweight mechanism to serve a business goal.
            - Pinterest example: user profiler, follower, feed, search, photo upload
        - Service Discovery: zookeeper help services find each other by keeping track of registered names, addresses, ports. Health check help verify integrity and are done using HTTP endpoint. use key-value store for config values
        - Application layer disadvantages:
            - loosly couple services, requires a different approach from an architectural, operations, and process viewpoint
            - increase complexity in terms of deployment and operations
    - Database
                     ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/Xkm5CXz.png)
        - Relational database management system
            - ACID
                - Atomic: each transaction is all or nothing
                - Consistent: any transaction bring the db from one valid state to another
                - Isolated: executing transactions concurrently has the same result as if the transactions were executed serially
                - Durable: once a transaction has been committed, it will remain so
            - Scale up
                - master/slave replication: master serves both reads and writes, replicating writes to slaves, which serve only reads. Slaves can replicate to additional slaves like tree. If master goes offline, the system can continue to operate in read-only mode until a slave is promoted to a master or a new master is provisioned.
                                     ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/C9ioGtn.png)
                    - master/slave replication disadvantages:
                        - need additional logic to promote a slave to a master
                        - replication disadvantages as below
                - master/master replication: both masters serve reads and writes and coordinate with each other on writes. If either master goes down, the system can continue to operate with both reads and writes.
                    ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/krAHLGg.png)
                    - master/master replication disadvantages:
                        - need a load balancer or change to application logic to determine where to write
                        - loosely consistent(violating ACID) or increased write latency due to synchronization
                        - conflict resolution are more needed as more write nodes are added and as latency increases
                        - replication disadvantages as below
                    - replication disadvantages:
                        - potential loss of data if master fails before any newly written data can be replicated to other nodes
                        - writes are replayed to the read replicas. if too many writes, the read replicas can get bogged down with replaying writes and can't do as many reads
                        - the more read slaves, the more need to replicate, lead to greater replication lag
                        - replication adds more hardware and additional complexity                                                     
                - federation: (or functional partitioning) splits up databases by function. for example, insteaf of a single, monolithic database, could have three: forums, users, and products, resulting in less read and write traffic to each database and therefor less replication lag. Smaller db result in more data that can fit in memory, which in turn results in more cache hits due to improved cache locality. With no single central master serializing writes, you can write in parallel, increasing the throughput.
                    ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/U3qV33e.png)
                    - federation disadvantages:
                        - not effective if schema requires huge functions or tables
                        - need to update application logic to determine which database to read and write
                        - joining data from 2 databases is more complex with a server link sp_addlinkedserver
                        - adds more hardware and additional complexity
                - sharding: sharding distributes data across different db such that each db can only manage a subset of the data. for example, as the number of users increase, more shards are added to the cluster. shard by users' last name initial or geographic location.
                ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/wU8x5Id.png)
                    - similar to the advantages of federation, sharding results in less read and write traffic, less replicatiion, and more cache hits. Index size is also reduced, which generally improve perf with faster queries. If one shard goes down, the other shardds are still operational, although replication is needed to avoid data loss. Also no single central master serializing writes allowing writing in parallel with increased throughput
                    - sharding disadvantages:
                        - need to update logic to work with shards, which result in complex SQL queries
                        - data distribution can be lopsided in a shard. A set of power users on a shard could result in increased load to that shard compared to others
                            - rebalancing add additional complexity, sharding based on consistent hashing can reduce teh amount of transferred data
                        - joining data from multiple shards is more complex
                        - sharding adds more hardware and complexity
                - denormmalization: improve read perf at the expense of some write perf. Redundant copies of the data are written in multiple tables to avoid expensive joins. Some RDBMS support materialized views which handle the work of storing redundant information and keeping redundant copies consistent.
                    - once data becomes distributed with tech such as federation and sharding, managing joins across data centers further increases complexity. Denormalization might avoid the need for such complex joins
                    - in most systems, read outnumber writes 100:1 or 1000:1. A read resulting in a complex db join can be very expensive, spending a significant amount of time on disk operations.
                    - denormalization disadvantages:
                        - data is duplicated
                        - constriants can help redundant copies of information stay in sync, which increases complexity of the database design
                        - a denormalized database under heavy write load might perform worse than its normalized counterpart
                - SQL tuning
                    - identify issues: benchmark and profile to simulate and uncover bottlenecks
                        - benchmark: simulate high-load situations with tools such as ab
                        - profile: enable tools such as the slow query log to help track perf issues
                    - Optimizations
                        - Tighten up the schema
                            - dumps to disk in contiguous blocks for fast access
                            - use CHAR instead of VARCHAR for fixed length fields
                                - CHAR allows for fast, random access, whereas with VARCHAR, you must find the end of a string before moving onto the next one
                            - use TEXT for large blocks of text such as blog posts
                                - TEXT allows for boolean searches. Using a TEXT field results in storing a pointer on disk that is used to locate the text block.
                            - use INT for large numbers up to 2^32 or 4 billion
                            - use DECIMAL for currency to avoid floating point representation errors
                            - avoid storing large BLOBS, store the location of where to get the object instead
                            - VARCHAR(255) is the largest number of characters that can be counted in an 8 bit number, often maximizing the use of a byte in some RDBMS
                            - Set NOT NULL constraint where applicable to improve search performance. especially if the null fields later updated to non-null.
                                - NULL values are not indexed in some DB, Oracle
                        - Use good indices
                            - SELECT, GROUP BY, ORDER BY, JOIN could be faster with indices
                            - indices are represented as self-balancing B-tree that keeps data sorted and allows searches, sequential access, insertions, deletions in logarithmic time
                            - Placing an index can keep data in memory, requiring more space
                            - Write could be slower since index need to be updated
                            - When loading large amounts of data, might be faster to disable indices, load the data, then rebuild the indices
                            
                        - Avoid expensive joins
                            - Denormalize where performance demands it
                        - Partition tables
                            - break up a table by putting hot spots in a seperate table to help keep it in memory
                        - Tune the query cache
                            - in some cases, query cache could lead to perf issue.
                                - high update rates
                                - large result sets
                                - dynamic queries
                                - data inconsistency
                                - limited cache size
        

## Reference
- [RabbitMQ Tutorials](https://web.archive.org/web/20230216005331/https://www.rabbitmq.com/tutorials/tutorial-one-python.html)
    

- [System Design Primer - System Design Topics](https://github.com/donnemartin/system-design-primer#system-design-topics-start-here)

- [Crack the system design interview](https://tianpan.co/notes/2016-02-13-crack-the-system-design-interview)
