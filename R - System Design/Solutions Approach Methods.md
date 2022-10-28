# Method 1

Step 1: Outline use cases, constraints and assumptions

Step 2: Create a high level design

Step 3: Design core components

Step 4: Scale the design

# Method 2

Step 1: Requirements clarification

Step 2: Back-of-the-envelope estimation

Step 3: System interface definition

Step 4: Defining data model

Step 5: High-level design (drawing)

Step 6: Detailed design

-   Dig into two or three major components (interviewer’s feedback should guide us to what parts of system needs further discussion)
-   Present different approaches with trade-offs for each approach
-   Example things to think about:
    -   How should we partition data to distribute data? Vertical partitioning? Horizontal partitioning?
    -   How much and at which layer should we introduce cache to speed things up?
    -   What components need better load balancing?

Step 7: Identifying and resolving bottlenecks

-   Is there a single point of failure in our system? How do we mitigate it?
    -   Horizontally scale to distribute traffic, add load balancers balance the distributed traffic, add redundancy to our data layer to mitigate data loss
-   Do we have enough replicas of the data so that if we lose a few servers, we can still serve our users?
-   How are we monitoring the performance of our service?