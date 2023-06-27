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
            - Disadvantages
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
        - Disadvantages
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
        - Disadvantages:
            - load balancer can be a performance bottle neck if it does not have enough resources or not configured properly
            - a single load balancer is a single point of failure, but configuring multiple load balancers further increase complexity
        
            
        

        
        
    - ACID
        - Atomic
        - Consistent
        - Isolated
        - Durable
        

## Reference
- [RabbitMQ Tutorials](https://web.archive.org/web/20230216005331/https://www.rabbitmq.com/tutorials/tutorial-one-python.html)
    

- [System Design Primer - System Design Topics](https://github.com/donnemartin/system-design-primer#system-design-topics-start-here)
