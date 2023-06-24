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
    -Consistency Patterns
        - Weak Consistency: after a write, read may or may not see it. Best effort only. Real time use cases: video chat may lose connection for some time
        - Eventual Consisitency: after a write, read will eventually see it. Data is replicated asyncronously. High availability systems: DNS, email.
        - Strong Consistency: after a write, read will see it. Data is replicated synchornously. Systems need transactions: file systems, RDBMS
        
        ![Alt Text](https://github.com/SitongChe/SystemDesign/blob/main/Consistancy%20Pattern.png?raw=true)
        
    - ACID
        - Atomic
        - Consistent
        - Isolated
        - Durable
        

## Reference
- [RabbitMQ Tutorials](https://web.archive.org/web/20230216005331/https://www.rabbitmq.com/tutorials/tutorial-one-python.html)
    

- [System Design Primer - System Design Topics](https://github.com/donnemartin/system-design-primer#system-design-topics-start-here)
