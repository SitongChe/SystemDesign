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
    - NoSQL: lack true ACID and favor eventual consistency
        - BASE
            - Basically available: the system guarantees availablity
            - Soft state: the state of the system may change over time, even without input
            - Eventual consistency: the system will become consistent over a period of time, given that the system doesn't receive input during that period
        - key-value store: hash table
            - O(1) read and write, backed by memory or SSD
            - maintain keys in lexicographic order, allowing efficient retrieval of key ranges, allowing for storing of metadata with a value
            - high performance and used for simple data models or rapidly changing data such as an in memory cache layer. complexity is shifted to the application layer if additional operations are needed
            - basis for more complex systems
        - document store: key-value store with documents stored as values
            - centered around documents XML JSON binary.
            - provid APIs or a query language to query based on the internal structure of the document itself. Note many key value stores include features for working with a value's metadata, blurring the lines between these two storage types.
            - documents are organized by collections, tags, metadata, or directories. Although documents can be organized or grouped together, documents may have fields that are completely different from each other.
            - Provide high flexibility and are often used for working with occasionally changing data.
        - wide column store: nested map ColumnFamily<RowKey, Columns<ColKey, Value, Timestamp>>
        ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/n16iOGk.png)
            - basic unit of data is a column (name/value pair). A column can be grouped in column families (analogous to a SQL table). Super column families further group column families. You can access each column independently with a row key, and columns with the same row key form a row. Each value contains a timestamp for versioning and for conflict resolution.
            - Google: Bigtable. Open source: HBase, often used in Hadoop ecosystem. Cassandra from Facebook. maintain keys in lexicographic order, allowing efficient retrieval of selective key ranges.
            - High availability, high scalability. often used for very large data sets.
        - graph database: graph
        ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/fNcl65g.png)
            - each node is a record and each arc(edge) is a relationship between two nodes. Graph databases are optimized to represent complex relationships with many foreign keys or many to many relationships
            - high performance for data models with complex relationships, such as social network. They are relatively new and not yet widely used.
            - REST APIs
    - SQL or NoSQL
       ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/wXGqG5f.png)
        - Reasons for SQL
            - Structured data
            - Strict schema
            - Relational data
            - Need for complex joins
            - Transactions
            - Clear patterns for scaling
            - More established: developers, community, code, tools, etc
            - Lookups by index are very fast
        - Reasons for NoSQL:
            - semi structured data
            - dynamic or flexible schema
            - non relational data
            - no need for complex joins
            - store many TB of data
            - very data intensive workload
            - very high throughput for IOPS
        - Sample data well suited for NoSQL:
            - rapid ingest of clickstream and log data
            - leaderboard or scoring data
            - temporary data such as a shopping cart
            - frequently accessed (hot) tables
            - metadata/lookup tables
    - Cache
    ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/Q6z24La.png)
        - Caching improve page load times and can reduce the load on your servers and databases. In this model, the dispatcher will first lookup if the request has been made before and try to find the previous result to return, in order to save the actual execution.
        - Databases often benefit from a uniform distribution of reads and writes across its partitions. Popular items can skew the distribution, causing bottlenecks. Putting a cache in front of a database can help absorb uneven loads and spikes in traffic
        - Client Cache. Caches can be located on the client side (OS or browser), server side, or in a distinct cache layer
        - CDN caching. CDNs are considered a type of cache.
        - Web server caching. Reverse proxies and caches such as Varnish can serve static and dynamic content directly. Web servers can also cache requests, returning responses without having to contact application servers.
        - Database caching. default config has some caching, optimized for a generic use case. Tweaking these settings for specific usage patterns can further boost performance.
        - Application caching. In memory caches such aas Memcached and Redis are key-value stores between your application and your data storage. Since the data is held in RAM, it is much faster than typical databases where data is stored in disk. RAM is more limited than disk, so cache invalidataion algorithms such as LRU can help invalidate cold entries and keep hot data in RAM.
            - Redis
                - Persistence option
                - Build-in data structures such as sorted sets and lists
            - 2 general categories: database queries and objects
                - row level
                - query level
                - fully formed serializable objects
                - fully rendered HTML
            - avoid file based caching, as it makes cloning and auto scaling more difficult
        - Caching at the database query level
            - hash the query as a key and store the result to the cache.
            - this approach suffers from expiration issues
                - hard to delete a cached result with complex queries
                - if one piece of data changes such as a table cell, need to delete all cached queries that might include the cached cell
        - Caching at object level
            - see the data as an object, similar to what you do with application code. Have your application assemble the dataset from the database into a class instance or a data structure
                - remove the object from cache if its underlying data has changed
                - allows for aync processing: workers assemble objects by consuming the latest cached object
                - what to cache:
                    - user sessions
                    - fully rendered web pages
                    - activity streams
                    - user graph data
        - When to update the cache
            - Cache aside
            ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/ONjORqk.png)
                - the application is responsible for reading and writing from storage, the cache does not interact with storage directly.
                - the application dose the following:
                    - look for entry in cache, resulting in a cache miss
                    - load entry from the database
                    - add entry to cache
                    - return entry

                - Memcached is generally used in this manner.
                - Subsequent reads of data added to cache are fast.
                - lazy loading. Only requested data is cached, which avoid filling up the cache with data that isn't requested
                - cache aside disadvantages:
                    - each cash miss results in three trips, which can cause a noticeable delay
                    - data can become stale if it is updated in the database. this issue is mitigated by setting a time to live (TTL), which forces an update of the cache entry, or by using write through
                    - when a node fails, it is replaced by a new empty node, increasing latency.
            - Write through
            ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/0vBc0hN.png)
                - the application uses the cache as the main data source, reading and writing data to it, while the cache is responsible for reading and writing to the database
                    - application adds/updates entry in cache
                    - cache synchronously writes entry to data store
                    - return
                    
            
                - write through is a slow operation due to write, but subsequent read of just written data are fast. Users are generally more tolerant of latency when updating data  than reading data. Data in cache is not stale.
                - write trhough disadvantages:
                    - when a new node is created due to failure or scaling. the new node will not cache entries until the entry is updated in the database. Cache aside in conjunction with write through can mitigated this issue.
                    - most data written might never be read, which can be minimized with a TTL.
            - Write behind (write back)
            ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/rgSrvjG.png)
                - in write behind, the app does the following:
                    - add/update entry in cache
                    - async write entry to the data store, improving write performance
                - write behind disadvantages:
                    - there could be data loss if the cache goes down prior to its contents hitting the data store
                    - it is mmore complex to implement write behind than it is to implement cache aside or write through
            - Refresh ahead
            ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/kxtjqgE.png)
                - configure the cache to automatically refresh any recently accessed cache entry prior to its expiration
                - refresh ahead can result in reduced latency vs read through if the cache can accurately predict which items are likely to be needed in the future
                - refresh ahead disadvantages:
                    - not accurately predicting which items are likely to be needed in the future can result in reduced performance than without refresh ahead
            - Cache Disadvantages:
                - need to maintain consistency between caches and the source of truth such as the database through cache invalidation
                - cache invalidation is a difficult problem, there is additional complexity associated with when to update the cache
                - need to make application changes such as adding Redis or memcached
    - Asynchronism
    ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/54GYsSx.png)
        - help reduce request times for expensive operations that would otherwise be performed in-line. They can also help by doing time consuming work in advance, such as aggregation of data
        - Message queues
            - receive, hold, and deliver messages. If an operation is too slow to perform inline, you can use a message queue with the following workflow:
                - an application publishes a job to the queue, then notifies the user of job status
                - a worker picks up the job from the queue, processes it, then signals the job is complete
            - the user is not blocked and the job is processed in the background. During this time, the client might optionally do a small amount of processing to make it seem like the task has completed. For example,if post a tweet, the tweet could be instantly posted to your timeline, but it could take some time before your tweet is actually delivered to all your followers.
            - Redis, RabbitMQ, Amazon SQS
        - Task queues
            - receive tasks and their related data. Runs them then delivers their results. They can support scheduling and can be used to run computationally intensive jobs in the background
            - Celery
        - Back pressure
            - if queue start to grow significantly, the queue size can become larger than memory. Resulting in cache misses, disk reads, and even slower performance. Back pressure can help by limiting the queue size, thereby maintaining a high throughput rate and good response times for jobs already in the queue. Once the queue fills up, clients get a server bussy or HTTP 503 code to try again later. Clients can retry the request at a later time, perhaps with exponential backoff.
        - Asynchronism disadvantages
            - use cases such as inexpensive calculations and realtime workflows might be better suited for synchronous operations, as introducing queues can add delays and complexity
    - Communication
    ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/5KeocQs.jpg)
        - HTTP
          - HTTP is an application layer protocol relying on lower-level protocols such as TCP and UDP.
          - A basic HTTP request consists of a verb (method) and a resource (endpoint).
          - Below are common HTTP verbs
        
        | Verb  | Description                                                  | Idempotent   | Safe | Cacheable               |
        |-------|--------------------------------------------------------------|--------------|------|-------------------------|
        | GET   | Reads a resource                                             | Yes          | Yes  | Yes                     |
        | POST  | Creates a resource or triggers a process that handles data   | No           | No   | Yes with freshness info |
        | PUT   | Creates or replaces a resource                               | Yes          | No   | No                      |
        | PATCH | Partially updates a resource                                 | No           | No   | Yes with freshness info |
        | DELETE| Deletes a resource                                           | Yes          | No   | No                      |
        
        Idempotent: An operation is idempotent if multiple identical requests have the same effect as a single request.
        
        - TCP
        
        ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/JdAsdvG.jpg)
        
          - Useful in applications that require high reliability but less time-critical, like web servers, database info, SMTP, FTP, SSH.
          - Use TCP over UDP when:
            - You need all of the data to arrive intact
            - You want to automatically make a best estimate use of the network throughput
      
        - UDP
          
        ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/yzDrJtA.jpg)

            - UDP is connectionless. packets are guaranteed only at packets level. may reach the destination out of order or not at all. no support for congestion control. but more efficient.
            - UDP can broadcast, sending datagrams to all devices on the subnet.
            - useful in real time use cases such as VoIP, video chat, streaming, realtime multiplayer games
            - use UDP over TCP when:
                - you need the lowest latency
                - late data is worse than loss of data
                - you want to implement your own error correction
    - Remote procedure call(PRC)
    ![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/iF4Mkb5.png)
        - Protobuf, thrift, avro
        - RPC is a request-response protocol:
            - client program - calls the client stub procedure. the parameters are pushed onto the stack like a local procedure call
            - client stub procedure - Marshals (packs) procedure id and arguments into a request message
            - client communication module - OS sends the message from the client to the server
            - server communication module - OS passes the incoming packets to the server stub procedure
            - server stub procedure - unmarshalls the results, calls the server procedure matching the procedure id and passes the given arguments
            - the server response repeats the steps above in reverse order
        - RPC is focused on exposing behaviors. RPCs are often used for perf reasons with internal communications, as you can hand craft native calls to better fit your use case
        - choose a native library (aka SDK) when:
            - you know your target platform
            - you want to control how your logic is accessed
            - you want to control how error control happens off your library
            - perf and end user experience is your primary concern
        - HTTP APIs following REST tent to be used more often for public APIs
        - RPC disadvantages:
            - RPC clients become tightly coupled to the service implementation
            - A new API must be defined for every new operation or use case
            - it can be difficult to debug RPC
            - might not be able to leverage existing tech out of the box. For example, it might require additional effort to ensure RPC calls are properly cached on caching servers such as Squid
    - Representational state transfer (REST)
        - REST is an carchitectural style enforcing a client/server model where the client acts on a set of resources managed by the server. The server provides a representation of resources and actions that can either manipulate or get a new representation of resources. All communication must be stateless and cacheable.
        - 4 qualities of a RESTful interface:
            - identify resources (URI in HTTP): use the same URI regardless of any operation
            - change the representations (Verbs in HTTP): user verbs, headers, and body
            - self descriptive error message (status response in HTTP): use status code, don't reinvent the wheel
            - HATEOAS (HTML interface in HTTP): your web service should be fully accessible in a browser
        - REST is focused on exposing data. It minimizes the coupling between client/server and is often used for public HTTP APIs. REST uses a more generic and uniform method of exposing resources through URIs, representation through headers, and actions through verbs such as GET, POST, PUT, DELETE, and PATCH. Being stateless, REST is great for horizontal scaling and partitioning.
        - REST Disadvantages:
            - focused on exposing data. not a good fit if resources are not naturally organized or accessed in a simple hierarchy. for example, returning all updated records from the past hour matching a particular set of events is not easily expressed as a path
            - REST relies on a few verbs, which sometimes doesn't fit the use case. for example, moving expired documents to the archive folder is not cleanly fit within the verbs.
            - fetching complicated resources with nested hierarchies requires multiple round trips between the client and server to render single views. e.g. fetching content of a blog entry and the comments on that entry.
            - over time, more fields might be added to an API response and older clients will receive all new data fields, even those that they don't need, as a result, it bloats the payload size and lead to larger latencies
            
        - RPC and REST calls comparison

| Operation                   | RPC                               | REST                               |
|-----------------------------|-----------------------------------|------------------------------------|
| Signup                      | POST /signup                      | POST /persons                      |
| Resign                      | POST /resign                      | DELETE /persons/1234               |
| Read a person               | GET /readPerson?personid=1234     | GET /persons/1234                  |
| Read a person’s items list  | GET /readUsersItemsList?personid=1234 | GET /persons/1234/items         |
| Add an item to a person’s items | POST /addItemToUsersItemsList | POST /persons/1234/items |
| Update an item              | POST /modifyItem                  | PUT /items/456                      |
| Delete an item              | POST /removeItem                  | DELETE /items/456                   |

        
    - Security
        - encrypt in transit and at rest
        - sanitize all user inputs or any input parameters exposed to user to prevent XSS and SQL injection
        - use parameterized queries to prevent SQL injection
        - use the principle of least privilege

## Real world architectures
![Alt Text](https://github.com/donnemartin/system-design-primer/blob/master/images/TcUo2fw.png)

## Appendix

- Powers of two table
  
| Power | Exact Value         | Approx Value   | Bytes   |
|-------|---------------------|----------------|---------|
| 7     | 128                 |                |         |
| 8     | 256                 |                |         |
| 10    | 1024                | 1 thousand     | 1 KB    |
| 16    | 65,536              |                | 64 KB   |
| 20    | 1,048,576           | 1 million      | 1 MB    |
| 30    | 1,073,741,824       | 1 billion      | 1 GB    |
| 32    | 4,294,967,296       |                | 4 GB    |
| 40    | 1,099,511,627,776   | 1 trillion     | 1 TB    |


- Latency numbers every programmer should know
  
| Task                                | Latency (ns)     | Latency (us)    | Latency (ms)   |
|-------------------------------------|------------------|-----------------|----------------|
| L1 cache reference                  | 0.5              |                 |                |
| Branch mispredict                   | 5                |                 |                |
| L2 cache reference                  | 7                |                 |                |
| Mutex lock/unlock                   | 25               |                 |                |
| Main memory reference               | 100              |                 |                |
| Compress 1K bytes with Zippy        | 10,000           | 10              |                |
| Send 1 KB bytes over 1 Gbps network | 10,000           | 10              |                |
| Read 4 KB randomly from SSD         | 150,000          | 150             |                |
| Read 1 MB sequentially from memory  | 250,000          | 250             |                |
| Round trip within same datacenter   | 500,000          | 500             |                |
| Read 1 MB sequentially from SSD     | 1,000,000        | 1,000           | 1              |
| HDD seek                            | 10,000,000       | 10,000          | 10             |
| Read 1 MB sequentially from 1 Gbps  | 10,000,000       | 10,000          | 10             |
| Read 1 MB sequentially from HDD     | 30,000,000       | 30,000          | 30             |
| Send packet CA->Netherlands->CA     | 150,000,000      | 150,000         | 150            |

Notes
-----
1 ns = 10^-9 seconds
1 us = 10^-6 seconds = 1,000 ns
1 ms = 10^-3 seconds = 1,000 us = 1,000,000 ns

![Alt Text](https://camo.githubusercontent.com/77f72259e1eb58596b564d1ad823af1853bc60a3/687474703a2f2f692e696d6775722e636f6d2f6b307431652e706e67)
               
        

## Reference
- [RabbitMQ Tutorials](https://web.archive.org/web/20230216005331/https://www.rabbitmq.com/tutorials/tutorial-one-python.html)
    

- [System Design Primer - System Design Topics](https://github.com/donnemartin/system-design-primer#system-design-topics-start-here)

- [Crack the system design interview](https://tianpan.co/notes/2016-02-13-crack-the-system-design-interview)
