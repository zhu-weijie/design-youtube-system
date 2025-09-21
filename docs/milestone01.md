### **Foundational Monolithic Service**

Problem:
Establish the most basic end-to-end flow: how can a user upload a video file and have it be playable?

Solution:
A single, monolithic API server. It will handle a direct HTTP POST request for upload, save the file to its local filesystem, and expose another endpoint to stream the file back to a user.

Trade-offs:
- Pro: Extremely simple to implement and understand. It provides a "walking skeleton" to validate the core concept.
- Con: This design is not scalable, has no data durability (a server failure results in data loss), and combines application logic with storage concerns, making it unsuitable for production.

### **Logical View (C4 Component Diagram)**

```mermaid
graph TD
    subgraph "Streamify System"
        api_server("
            <b>API Server</b>
            <br>
            Handles video uploads,
            <br>
            stores files locally,
            <br>
            and streams them back.
        ")
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
            direction TB
            container("
                <b>Docker Container</b>
                <br>
                API Server Application
            ")
        end
        
        ebs("<b>EBS Volume</b><br>(Local Filesystem)")

        container -- "Stores/Reads Video File" --> ebs
    end
    
    browser -- "HTTP Requests" --> container
```

### **Component-to-Resource Mapping Table**

| Logical Component | Physical Resource                                                                             | Rationale                                                                                                       |
| :---------------- | :-------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------- |
| API Server        | A Docker container running on a single AWS EC2 Instance with an attached Elastic Block Store (EBS) volume. | This represents the simplest possible deployment to create a functional prototype. The EBS volume provides basic persistent storage for the single node. |
