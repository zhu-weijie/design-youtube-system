### **Logical View (C4 Component Diagram)**

```mermaid
graph TD
    subgraph "Streamify System"
        uploads_storage("<b>Uploads Storage</b>")

        subgraph "Video Processing Pipeline (DAG)"
            orchestrator("<b>Workflow Orchestrator</b>")

            subgraph "Stage 1: Split"
                splitter("<b>Video Splitter Task</b>")
            end

            subgraph "Stage 2: Parallel Transcode (Fan-Out)"
                segment_transcoder("
                    <b>Segment Transcoder Task</b>
                    <br>
                    (Many Parallel Instances)
                ")
            end

            subgraph "Stage 3: Assemble (Fan-In)"
                assembler("<b>Manifest Assembler Task</b>")
            end
        end

        transcoded_storage("<b>Transcoded Media Storage</b>")

        %% Define Workflow
        uploads_storage -- "Triggers Workflow" --> orchestrator
        
        orchestrator -- "Invokes" --> splitter
        splitter -- "Reads Original Video" --> uploads_storage

        orchestrator -- "Invokes Parallel Tasks" --> segment_transcoder
        segment_transcoder -- "Reads Original Segment" --> uploads_storage
        segment_transcoder -- "Writes Transcoded Segment" --> transcoded_storage
        
        segment_transcoder -- "Reports Completion" --> orchestrator
        
        orchestrator -- "Invokes on Completion" --> assembler
        assembler -- "Writes Manifest" --> transcoded_storage
    end
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

        step_functions("
            <b>AWS Step Functions</b>
            <br>
            (Transcoding State Machine)
        ")

        lambda_splitter("
            <b>AWS Lambda</b>
            <br>
            (Splitter Function)
        ")

        lambda_transcoder("
            <b>AWS Lambda</b>
            <br>
            (Parallel Transcoder Functions)
        ")
        
        lambda_assembler("
            <b>AWS Lambda</b>
            <br>
            (Assembler Function)
        ")

        s3_transcoded("
            <b>S3 Bucket</b>
            <br>
            (transcoded-media)
        ")

        %% Control Flow
        s3_uploads -- "S3 Event triggers" --> step_functions
        step_functions -- "Invokes Splitter" --> lambda_splitter
        step_functions -- "Invokes Transcoders (Fan-Out)" --> lambda_transcoder
        step_functions -- "Invokes Assembler (Fan-In)" --> lambda_assembler

        %% Data Flow
        lambda_splitter -- "Reads Original" --> s3_uploads
        lambda_transcoder -- "Reads Original Segments" --> s3_uploads
        lambda_transcoder -- "Writes Transcoded Segments" --> s3_transcoded
        lambda_assembler -- "Writes Manifest" --> s3_transcoded
    end
```

### **Component-to-Resource Mapping Table**

| Logical Component         | Physical Resource                                 | Rationale                                                                                                                                                                                                                                                         |
| :------------------------ | :------------------------------------------------ | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Workflow Orchestrator     | AWS Step Functions                                | Step Functions is a serverless workflow orchestrator designed specifically for managing complex, stateful DAGs. It handles state, error handling, and parallel execution ("Map State"), making it a perfect fit for our fan-out/fan-in transcoding model. |
| Video Splitter Task       | AWS Lambda Function                               | Splitting a video file is a short-lived, event-driven task. Lambda provides a cost-effective, serverless compute model that can scale instantly without managing servers.                                                                                    |
| Segment Transcoder Task   | AWS Lambda Function                               | Transcoding small segments is an ideal use case for Lambda. We can run thousands of parallel Lambda invocations simultaneously, one for each segment/resolution pair, achieving massive parallelization without managing any EC2 instances.          |
| Manifest Assembler Task   | AWS Lambda Function                               | Generating a text-based manifest file is a very fast operation, making it a perfect final step in the Lambda-based workflow.                                                                                                                                     |
