### **Logical View (C4 Component Diagram)**

```mermaid
graph TD
    subgraph "Streamify System"
        api_server("
            <b>API Server</b>
            <br>
            Control plane for uploads.
        ")

        uploads_storage("
            <b>Uploads Storage</b>
            <br>
            Stores original video files.
        ")

        message_queue("
            <b>Message Queue</b>
            <br>
            Decouples upload from transcoding.
        ")

        transcoding_worker("
            <b>Transcoding Worker</b>
            <br>
            Processes videos into multiple formats.
        ")
        
        transcoded_storage("
            <b>Transcoded Media Storage</b>
            <br>
            Stores processed video files.
        ")
        
        uploads_storage -- "Triggers Notification" --> message_queue
        message_queue -- "Sends Job" --> transcoding_worker
        transcoding_worker -- "Reads Original File" --> uploads_storage
        transcoding_worker -- "Writes Processed Files" --> transcoded_storage
    end

    user("<b>User</b>")

    user -- "Requests Pre-Signed URL" --> api_server
    api_server -- "Returns Pre-Signed URL" --> user
    user -- "Uploads Video File" --> uploads_storage
```

### **Physical View (AWS Deployment Diagram)**

```mermaid
graph TD
    subgraph "AWS Cloud"
        s3_uploads("
            <b>S3 Bucket</b>
            <br>
            (raw-uploads)
        ")

        sqs("
            <b>SQS Queue</b>
            <br>
            (transcoding-jobs)
        ")

        subgraph "Transcoding Worker Fleet (Auto Scaling Group)"
            worker_instance("
                <b>EC2 Instance</b>
                <br>
                Runs Transcoding App
            ")
        end

        s3_transcoded("
            <b>S3 Bucket</b>
            <br>
            (transcoded-media)
        ")
        
        %% Define the process flow
        s3_uploads -- "S3 Event Notification" --> sqs
        sqs -- "Provides Job" --> worker_instance
        worker_instance -- "Reads Original Video" --> s3_uploads
        worker_instance -- "Writes Transcoded Videos" --> s3_transcoded
    end
```

### **Component-to-Resource Mapping Table**

| Logical Component        | Physical Resource                                                                | Rationale                                                                                                                                                                                                                         |
| :----------------------- | :------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Uploads Storage          | AWS S3 Bucket (e.g., `raw-uploads`)                                                  | A dedicated bucket to store the original, unmodified video files.                                                                                                                                                                 |
| Message Queue            | AWS Simple Queue Service (SQS)                                                     | SQS is a fully managed message queuing service that is perfect for decoupling services. S3 Event Notifications can integrate directly with it, making it an ideal choice to trigger the asynchronous workflow reliably.                 |
| Transcoding Worker       | An Auto Scaling Group of EC2 Instances running the transcoding application.      | A separate fleet of compute resources is needed for the CPU-intensive transcoding tasks. An Auto Scaling Group allows the fleet to grow or shrink based on the workload (i.e., the number of messages in the SQS queue), optimizing costs. |
| Transcoded Media Storage | AWS S3 Bucket (e.g., `transcoded-media`)                                             | A separate bucket to store the output of the transcoding process. This separation keeps the original files safe and simplifies access control and lifecycle management for the different types of data.                               |
