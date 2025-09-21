### **Implement Direct Uploads via Pre-Signed URLs**

Problem:
Routing gigabytes of video data through the API server consumes its network bandwidth and CPU, preventing it from handling other requests efficiently. This makes the API server a severe bottleneck and limits the scalability of the entire system.

Solution:
Shift to a direct-to-storage upload pattern using pre-signed URLs. The client first makes a lightweight API call to the server to signal its intent to upload. The API server authenticates the request, generates a temporary, secure pre-signed URL for the object storage, and returns it to the client. The client then uses this URL to upload the video file directly to the object storage service, completely bypassing the API server for the heavy data transfer.

Trade-offs:
- Pro: Massively improves the scalability of the API server (NFR3) by offloading the data transfer work. The server is freed up to handle a much higher volume of actual API requests.
- Con: This pattern introduces more complexity on the client-side, which now needs to handle a two-step upload process.

### **Logical View (C4 Component Diagram)**

```mermaid
graph TD
    subgraph "Streamify System"
        api_server("
            <b>API Server</b>
            <br>
            Control plane for uploads.
            <br>
            Proxies video for streaming.
        ")

        storage("
            <b>Object Storage</b>
            <br>
            Receives direct uploads from users.
            <br>
            Stores video files.
        ")
        
        api_server -- "Reads Video for Streaming" --> storage
    end

    user("<b>User</b>")

    %% Upload Flow
    user -- "Requests Pre-Signed URL" --> api_server
    api_server -- "Returns Pre-Signed URL" --> user
    user -- "Uploads Video File Directly" --> storage
    
    %% Playback Flow
    user -- "Requests Video Stream" --> api_server
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

        container -- "Generates URL via AWS SDK" --> s3
    end
    
    browser -- "API Request (Lightweight)" --> container
    browser -- "Uploads Video File (Heavy)" --> s3
```

### **Component-to-Resource Mapping Table**

| Logical Component | Physical Resource                                     | Rationale                                                                                                                                                                                            |
| :---------------- | :---------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| API Server        | A Docker container running on an AWS EC2 Instance.      | The API server's role shifts to a **control plane** for uploads. It no longer handles the video data stream during upload, only a lightweight request to authorize the action and generate a secure, temporary URL. |
| Object Storage    | AWS Simple Storage Service (S3)                         | S3's pre-signed URL feature is used to enable secure, direct-from-client uploads. This offloads the data transfer burden from our application servers, allowing the system to scale upload and API traffic independently. |
