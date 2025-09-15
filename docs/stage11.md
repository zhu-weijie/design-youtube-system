### **Add a Metadata Caching Layer**

Problem:
When a video becomes popular or goes viral, the API service receives a massive number of concurrent read requests for the same metadata (e.g., title, description, manifest URL). Routing all these requests to the primary database can create a "hotspot," overwhelming the database with redundant queries. This can lead to high latency, poor application performance, and potentially bring down the database, causing a site-wide outage.

Solution:
Introduce an in-memory caching layer between the API service and the metadata database. The data access pattern will be "cache-aside":
1.  **Request:** The API service receives a request for video metadata.
2.  **Cache-Check (Hit):** It first attempts to fetch the data from the in-memory cache. If the data exists (a cache hit), it is returned immediately with sub-millisecond latency.
3.  **Cache-Check (Miss):** If the data does not exist in the cache (a cache miss), the API service then queries the primary database.
4.  **Populate & Return:** The service returns the data to the user and simultaneously writes the data into the cache with a defined Time-to-Live (TTL), so subsequent requests will be served from the cache.

Trade-offs:
- Pro: Dramatically improves read latency for frequently accessed data. Protects the primary database from read-heavy workloads, significantly increasing the overall stability and scalability of the system.
- Con: Introduces another component that must be managed and monitored. Creates a possibility of stale data (data is updated in the database but the old version remains in the cache until its TTL expires), which requires a cache invalidation strategy for sensitive updates.

### **Logical View (C4 Component Diagram)**

```mermaid
graph TD
    subgraph "Streamify System"
        api_service("
            <b>API Service</b>
            <br>
            Handles API requests.
            <br>
            Implements cache-aside logic.
        ")

        cache("
            <b>Cache</b>
            <br>
            Stores hot video metadata.
        ")

        database("
            <b>Metadata Database</b>
            <br>
            (Source of Truth)
            <br>
            Stores all video metadata.
        ")

        api_service -- "Reads from / Writes to" --> cache
        api_service -- "Reads from (on Cache Miss)" --> database
    end

    user("<b>User</b>")
    user -- "API Requests" --> api_service
```

### **Physical View (AWS Deployment Diagram)**

```mermaid
graph TD
    subgraph "AWS Cloud"
        alb("
            <b>Application Load Balancer</b>
        ")

        subgraph "ECS Service (Auto Scaling)"
            api_task("<b>ECS Task</b><br>(API Container)")
        end
        
        redis("
            <b>ElastiCache for Redis</b>
            <br>
            (In-Memory Cache Cluster)
        ")

        rds("
            <b>RDS Database</b>
            <br>
            (e.g., PostgreSQL)
        ")
        
        alb --> api_task
        api_task -- "GET/SET (Cache)" --> redis
        api_task -- "SELECT (on Cache Miss)" --> rds
    end
```

### **Component-to-Resource Mapping Table**

| Logical Component   | Physical Resource                                                              | Rationale                                                                                                                                                                                                                                                                       |
| :------------------ | :----------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| API Service         | An AWS ECS Service with an ALB.                                                  | The API container's application logic is now updated to include the cache-aside pattern, connecting to both Redis and RDS clients to manage data retrieval.                                                                                                                   |
| **Cache**           | **Amazon ElastiCache for Redis**                                                 | **Redis is an extremely fast in-memory key-value store, making it the industry standard for caching. ElastiCache is the managed AWS offering, which handles the operational overhead of running a Redis cluster, providing high availability and scalability for our cache.** |
| **Metadata Database** | **Amazon RDS for PostgreSQL**                                                  | **RDS is a managed relational database service. It provides the durability, consistency, and transactional integrity required for our primary metadata store (video titles, user info, etc.). Using a managed service like RDS offloads database administration tasks.** |

### Overall Logical View (C4 Component Diagram)

```mermaid
graph TD
    subgraph "User's Device"
        direction LR
        uploader("<b>Uploader Client</b><br>Manages resumable uploads")
        video_player("<b>Video Player</b><br>Handles adaptive streaming")
    end

    subgraph "Streamify System"
        api_service("
            <b>API Service</b>
            <br>Handles metadata & authentication
        ")

        subgraph "Data Persistence"
            direction LR
            cache("<b>Cache</b><br>Hot Metadata")
            database("<b>Database</b><br>All Metadata")
        end

        uploads_storage("
            <b>Uploads Storage</b>
            <br>Stores original videos
        ")
        
        processing_pipeline("
            <b>Video Processing Pipeline</b>
            <br>(Asynchronous DAG Workflow)
        ")

        transcoded_storage("
            <b>Transcoded Media Storage</b>
            <br>(CDN Origin)
        ")
        
        cdn("
            <b>CDN</b>
            <br>Global Media Cache
        ")
    end

    %% User Flows
    uploader -- "Control: Initiate/Complete Upload" --> api_service
    uploader -- "Data: Uploads Chunks" --> uploads_storage
    
    video_player -- "Requests Metadata/URL" --> api_service
    video_player -- "Fetches Media Segments" --> cdn

    %% Internal System Flows
    api_service -- "Cache-Aside Logic" --> cache
    api_service -- "Reads on Miss" --> database
    
    uploads_storage -- "Triggers" --> processing_pipeline
    processing_pipeline -- "Reads Original" --> uploads_storage
    processing_pipeline -- "Writes Transcoded Media" --> transcoded_storage
    
    cdn -- "Origin Pull on Miss" --> transcoded_storage
```

### **Overall Physical View (AWS Deployment Diagram)**

```mermaid
graph TD
    %% --- User's Device ---
    subgraph "User's Device"
        direction LR
        uploader("<b>Uploader Library</b><br>(e.g., Uppy)")
        player("<b>Video Player Library</b><br>(e.g., HLS.js)")
    end

    %% --- AWS Cloud ---
    subgraph "AWS Cloud"
        %% API & Persistence are closely linked
        subgraph "API & Data Persistence"
            direction TB
            alb("<b>Application Load Balancer</b>")
            ecs_api("<b>ECS Service (Fargate)</b><br>API Container Tasks")
            
            subgraph "Persistence Layer"
                direction LR
                redis("<b>ElastiCache for Redis</b>")
                rds("<b>RDS Database</b>")
            end
            
            alb --> ecs_api
            ecs_api -- "Cache-Aside Logic" --> redis
            ecs_api -- "Reads/Writes" --> rds
        end

        %% Processing is a distinct flow
        subgraph "Asynchronous Video Processing"
            direction TB
            s3_uploads("<b>S3 Bucket (raw-uploads)</b>")
            sfn("<b>AWS Step Functions (DAG)</b>")
            s3_uploads -- "S3 Event Triggers" --> sfn
        end

        %% Delivery is a distinct flow
        subgraph "Content Delivery Network (CDN)"
            direction TB
            s3_transcoded("<b>S3 Bucket (transcoded-media / Origin)</b>")
            cloudfront("<b>CloudFront Distribution</b>")
            cloudfront -- "Origin Pull" --> s3_transcoded
        end
    end

    %% --- Interactions between Subgraphs ---
    
    %% Upload Flow
    uploader -- "Control: Initiate/Sign/Complete" --> alb
    uploader -- "Data: Uploads Chunks" --> s3_uploads

    %% Playback Flow
    player -- "API: Get Metadata/URL" --> alb
    player -- "Data: Fetches Media" --> cloudfront
    
    %% Internal Processing Flow
    sfn -- "Orchestrates workers to read from" --> s3_uploads
    sfn -- "Orchestrates workers to write to" --> s3_transcoded
```
