### **Logical View (C4 Component Diagram)**

```mermaid
graph TD
    subgraph "Streamify System"
        api_server("
            <b>API Server</b>
            <br>
            Acts as a proxy for video I/O.
        ")

        storage("
            <b>Object Storage</b>
            <br>
            Provides durable, scalable
            <br>
            storage for video files.
        ")

        api_server -- "Writes/Reads Video Files" --> storage
    end

    user("<b>User</b>")

    user -- "Uploads Video (HTTP POST)" --> api_server
    api_server -- "Streams Video (HTTP GET)" --> user
```

### **Physical View (AWS Deployment Diagram)**

```mermaid
graph TD
    subgraph "User's Device"
        browser("<b>Browser</b>")
    end

    subgraph "AWS Cloud"
        subgraph "EC2 Instance"
            container("
                <b>Docker Container</b>
                <br>
                API Server Application
            ")
        end
        
        s3("
            <b>S3 Bucket</b>
            <br>
            Durable Object Storage
        ")

        container -- "Reads/Writes via AWS SDK" --> s3
    end
    
    browser -- "HTTP Requests" --> container
```

### **Component-to-Resource Mapping Table**

| Logical Component | Physical Resource                                     | Rationale                                                                                                                                                             |
| :---------------- | :---------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| API Server        | A Docker container running on a single AWS EC2 Instance. | The API server remains the single entry point. Its deployment model is unchanged for now.                                                                            |
| Object Storage    | AWS Simple Storage Service (S3)                         | S3 is the industry standard for object storage, offering 11 nines of durability, virtually infinite scalability, and a cost-effective pay-as-you-go pricing model. It perfectly matches our non-functional requirements for durability and scalability. |
