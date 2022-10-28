Source: [](https://akshay-iyangar.github.io/system-design/)[https://akshay-iyangar.github.io/system-design/](https://akshay-iyangar.github.io/system-design/)

## Key Characteristics of Distributed Systems

-   Scalability
    -   Ability to handle growing demand on a system
    -   Many factors play into the growing demand
        -   Increased data volume
        -   Increased amount of work
    -   Horizontal vs Vertical Scaling
        -   Horizontal: increase number of machines used in pool of resources
        -   Vertical: increase power on existing machines (eg: increase RAM, more CPU cores, faster networking, more data storage)
-   Reliability
    -   The likelihood that a system will fail
    -   A distributed system is considered reliable if it continues delivering its services despite system failures
    -   Typically, a system provides redundancy of software components and data to mitigate failures
-   Availability
    -   SLAs (Service Level Agreements) are a way of quantifying availability of a system
        -   eg: 99.9% uptime of a system in a given month would translate to 43m 12s of downtime in a month
    -   If a system is reliable, it is also available. However, a system that is available may not necessarily be reliable
-   Efficiency
    -   Two standard measures:
        -   response time (latency): delay to receive initial response (eg: 50ms TTFB)
        -   throughput (bandwidth): volume of requests handled in a given time unit (eg: 10M messages / second)
-   Manageability
    -   how easy it is to operate and maintain a system
    -   things to consider for manageability:
        -   how easy to triage and understand problems as they occur
        -   how easy it is to make changes to system
        -   how easy is it to operate

## Load Balancing

-   A typical component of a distributed system
-   Incorporating into system provides reliability and availability of the system
-   Its function is to spread traffic across multiple backends
-   Typically sits between the client and server
-   We can balance load at each layer of a system
    -   between user and server
    -   between server and platform layer: application servers or cache servers
    -   between platform layer and database
-   benefits
    -   faster, uninterrupted service
    -   service providers have less downtown, full server failure won’t affect end user experience since LB will redirect traffic to healthy servers
    -   smart balancers can preemptively identify bottlenecks before they happen
    -   fewer failed or stressed components
-   LB uses health checks on backend servers to determine which servers to send traffic to
-   LB can be configured to use different load balancing methods depending on use case
-   LBs can also be redundant as a single LB can be a point of failure in a system; each LB is configured to periodically run health checks on the other LB

## Caching

-   caching takes advantage of idea that requested data will likely be requested again
    
-   Application server cache
    
    -   caching can exist directly on the request layer node
    -   there can be consistency issues if an LB randomly distributes requests across nodes
        -   this can be addressed by having a global cache or distributed caches
-   CDN
    
    -   useful caching for serving static media
    -   can serve static media from a separate domain using Nginx for system isn’t large enough yet
    -   if data isn’t available, CDN will query from backend servers
-   Cache Invalidation
    
    -   requires maintenance when implementing caches; consistency between cache and database storage should be factored
    -   Write-through
        -   2 writes happen on the cache and the DB
        -   Pro: data consistency between the cache and data store
        -   Con: increased latency due to 2 writes happening for every request
    -   Write-around
        -   data is directly written to data store and goes “around” the cache
        -   Pro: reduces cache write operations to data store that won’t be read immediately
        -   Con: increased latency due to required read from slower data stores
    -   Write-back
        -   data is written to cache only
        -   permanent storage write operation only after certain condition is met (eg: time interval)
        -   Pro: low latency and high throughput for write-intensive applications
        -   Con: speed comes at cost of risk in data loss in case of a crash since data may only exist on cache at a given time
    -   Cache Eviction Policies
        -   FIFO (stack): discards first item accessed
        -   LIFO (queue): discards item access most recently first
        -   LRU: discards least recently used items first
        -   MRU: discards most recently used items first
        -   LFU: maintains count of how often item is used, discards those that are least often used
        -   RR: random
    
    ## Data Partitioning