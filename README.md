<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <link rel="stylesheet" href="mermaid.min.css">
</head>
<body>
  <div class="mermaid">
    sequenceDiagram
      participant User
      participant Client
      participant API Server
      participant Queue
      participant DynamoDB Connection
      participant Activity Poller
      participant Worker Manager
      participant Worker Helper
      participant Worker
      participant AWS SFN
      participant AWS Batch
      participant AWS DynamoDB
      participant AWS Queue
      participant AWS S3
      participant AWS ElastiCache
      Client-->>+API Server:  Register Worker POST /v2/definitions
      API Server-->>+AWS Batch:  Register Worker Batch Job Definition
      AWS Batch-->>-API Server: Batch Job Definition ARN
      API Server-->>+DynamoDB Connection: Add Resource
      DynamoDB Connection-->>AWS DynamoDB: Update DB Resource
      DynamoDB Connection-->>-API Server: Return Worker URN
      API Server-->>-Client: POST /v2/definitions = Worker URN 
      Client-->>+API Server:  Register Job POST /v2/definitions
      API Server-->>+AWS SFN:  Register ASL Job Definition
      AWS SFN-->>-API Server: SFN Machine ARN
      API Server-->>+DynamoDB Connection: Add Resource
      DynamoDB Connection-->>AWS DynamoDB: Update DB Resource
      DynamoDB Connection-->>-API Server: Return Job URN
      API Server-->>-Client: POST /v2/definitions = Job URN 
      User-->>+API Server: POST /v2/jobs/requests
      API Server-->>AWS Queue: Send Request Message
      API Server-->>-User: Request URN
      AWS Queue-->>+Queue: Receive Message
      Note over Queue: Process Job Request
      Queue-->>+AWS SFN: Start Execution
      AWS SFN-->>-Queue: Execution ARN
      Note over Queue: Create Job Resource
      Queue-->>DynamoDB Connection: Add Job Resource
      Queue-->>DynamoDB Connection: Update Request Resource With Job URN
      Queue-->>-AWS Queue: Delete Request Message
      User-->>+API Server: GET /v2/jobs/requests/{urn}
      API Server-->>+DynamoDB Connection: Get Resource
      DynamoDB Connection-->>AWS DynamoDB: Get Resource
      DynamoDB Connection-->>-API Server: Return Job Request Resource
      API Server-->>-User: Request FULFILLED
      Note over User: Note down Job URN
      User-->>+API Server: GET /v2/jobs/{urn}
      API Server-->>+DynamoDB Connection: Get Resource
      DynamoDB Connection-->>AWS DynamoDB: Get Resource
      DynamoDB Connection-->>-API Server: Return Job Resource
      API Server-->>-User: Job Status = RUNNING
      loop Activity Poller
        Activity Poller-->>+AWS SFN: Poll for Activity
        AWS SFN-->>- Activity Poller: Activity Task Received
        Note over  Activity Poller: Create New </br>Worker Info
        Activity Poller-->>+AWS Batch: Submit Worker Job
        AWS Batch-->>- Activity Poller: Get Batch Job ID
        Activity Poller-->>AWS ElastiCache: Cache Worker Info Resource
        Note over  Activity Poller: Update Parent </br>Job Information
        Activity Poller-->>DynamoDB Connection: Update Job Resource
        DynamoDB Connection-->>AWS DynamoDB: Update Job Resource
      end
      User-->>+API Server: GET /v2/jobs/{urn}
      API Server-->>+DynamoDB Connection: Get Resource
      DynamoDB Connection-->>AWS DynamoDB: Get Resource
      DynamoDB Connection-->>-API Server: Return Job Resource
      API Server-->>-User: Job Status = RUNNING
      Worker-->>+Worker Manager: POST /v2/workers/start
      Worker Manager-->>AWS ElastiCache: Get Worker Info From Cache
      Worker Manager-->>-Worker: Start Payload
      Note over Worker: Process Input
      Worker-->>+Worker Helper: POST /v2/workers/files
      Worker Helper-->>AWS ElastiCache: Get Worker Info From Cache
      Worker Helper-->>AWS S3: Put File on S3
      Worker Helper-->>-Worker: Object Key
      Worker-->>+Worker Helper: GET /v2/workers/files/{key}
      Worker Helper-->>AWS ElastiCache: Get Worker Info From Cache
      Worker Helper-->>AWS S3: Get File From S3
      AWS S3-->>-Worker: File, Presigned URL
      Note over Worker: Process File
      Note over Worker: Compute Results
      Worker-->>+Worker Helper: POST /v2/workers/files
      Worker Helper-->>AWS ElastiCache: Get Worker Info From Cache
      Worker Helper-->>AWS S3: Put File on S3
      Worker Helper-->>-Worker: Object Key
      Worker-->>+Worker Manager: POST /v2/worker/results
      Worker Manager-->>AWS ElastiCache: Get Worker Info From Cache
      Note over Worker Manager: Update Worker Results
      Worker Manager-->>AWS ElastiCache: Cache Worker Info Resource
      Note over  Worker Manager: Update Parent Job Information
      Worker Manager-->>DynamoDB Connection: Update Job Resource
      DynamoDB Connection-->>AWS DynamoDB: Update Job Resource
      Worker Manager-->>-Worker: AddResults Success
      Worker-->>+Worker Manager: POST /v2/worker/finish
      Worker Manager-->>AWS ElastiCache: Get Worker Info From Cache
      Note over Worker Manager: Update Worker </br>Status
      Worker Manager-->>AWS ElastiCache: Cache Worker Info Resource
      Note over  Worker Manager: Update Parent </br>Job Information
      Worker Manager-->>DynamoDB Connection: Update Job Resource
      DynamoDB Connection-->AWS DynamoDB: Update Job Resource
      Worker Manager-->>Queue: Send Update Job Message
      Worker Manager-->>-Worker: Finish Success
      Queue-->>+AWS Queue: Receive Message
      Queue-->>AWS SFN: Get Execution Status
      Queue-->>DynamoDB Connection: Update Job Resource
      DynamoDB Connection-->>AWS DynamoDB: Update Job Resource
      Queue-->>-AWS Queue: Delete Update Job Message
      User-->>+API Server: GET /v2/jobs/{urn}
      API Server-->>+DynamoDB Connection: Get Resource
      DynamoDB Connection-->>AWS DynamoDB: Get Resource
      DynamoDB Connection-->>-API Server: Return Job Resource
      API Server-->>-User: Job Status = RUNNING
      Note over User: Access Intermediate </br>Results
      User-->>+API Server: GET /v2/files/{key}
      API Server-->>AWS S3: Get Resource
      AWS S3-->>-User: Result File, Presigned URL
      Note over Client,AWS ElastiCache:    
      Note over Client,AWS ElastiCache:    
      Note over Client,AWS ElastiCache:    
      Note over Client,AWS ElastiCache:    
      Worker-->>+Worker Manager: POST /v2/worker/finish
      Worker Manager-->>AWS ElastiCache: Get Worker Info From Cache
      Note over Worker Manager: Update Worker </br>Status
      Worker Manager-->>AWS ElastiCache: Cache Worker Info Resource
      Note over  Worker Manager: Update Parent </br>Job Information
      Worker Manager-->>DynamoDB Connection: Update Job Resource
      DynamoDB Connection-->>AWS DynamoDB: Update Job Resource
      Worker Manager-->>Queue: Send Update Job Message
      Worker Manager-->>-Worker: Finish Success
      Queue-->>+AWS Queue: Receive Message
      Queue-->>AWS SFN: Get Execution Status (COMPLETED)
      Note over Queue: Job Status </br>COMPLETED
      Queue-->>DynamoDB Connection: Update Job Resource
      DynamoDB Connection-->>AWS DynamoDB: Update Job Resource
      Queue-->>-AWS Queue: Delete Update Job Message
      User-->>+API Server: GET /v2/jobs/{urn}
      API Server-->>+DynamoDB Connection: Get Resource
      DynamoDB Connection-->>AWS DynamoDB: Get Resource
      DynamoDB Connection-->>-API Server: Return Job Resource
      API Server-->>-User: Job Status = COMPLETED
      Note over User: Access Intermediate </br>Results
      User-->>+API Server: GET /v2/files/{key}
      API Server-->>AWS S3: Get Resource
      AWS S3-->>-User: Result File, Presigned URL
  </div>
  <script src="mermaid.min.js"></script>
  <script>mermaid.initialize({startOnLoad:true});</script>
  <style type="text/css">
    g rect .actor {
      stroke: #ccf;
      fill: #ec0000;
    }
  </style>
</body>
</html>
