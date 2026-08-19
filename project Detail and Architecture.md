
---

- a queue orchestration platform that manages multiple independent queues belonging to multiple consuming applications.

#### The Relationship

```
                    QueueLess
                        │
       ┌────────────────┼────────────────┐
       │                │                │
     App A            App B            App C
       │                │                │
   ┌───┼───┐        ┌───┼───┐          │
   ▼   ▼   ▼        ▼   ▼   ▼          ▼
  Q1  Q2  Q3       Q1  Q2  Q3          Q1
```


#### The hierarchy

```
QueueLess
   │
   ├── Consumer/Application A
   │      ├── Queue A1
   │      │     ├── Job
   │      │     ├── Job
   │      │     └── Job
   │      │
   │      ├── Queue A2
   │      │     ├── Job
   │      │     └── Job
   │      │
   │      └── Queue A3
   │
   ├── Consumer/Application B
   │      ├── Queue B1
   │      └── Queue B2
   │
   └── Consumer/Application C
          └── Queue C1
```


#### Each Queue has its own configuration

```
Queue A1
  Strategy: FIFO
  Capacity: 10

Queue A2
  Strategy: PRIORITY
  Capacity: 5

Queue A3
  Strategy: ADAPTIVE
  Capacity: 20
```


---
It is a generic service that other applications can integrate with when they have work that needs:
- queuing
- scheduling
- prioritization
- waiting-time estimation
- dynamic resource allocation
- real-time status updates
- adaptive scheduling

==An application gives QueueLess a piece of work, and QueueLess decides how and when that work should be processed.

---

### Flow with respect to the Consumer Application

- QueueLess is a standalone queue orchestration service designed to be integrated into microservice-based or monolithic applications.
- QueueLess does'nt perform the actual work.

- it decides when and how the job should be processed. 

```Flow
Application
     |
     | "I have this job"
     ↓
QueueLess
     |
     | manages queue
     ↓
QueueLess says:
"Job is ready"
     |
     ↓
Application
     |
     | actually processes job
     ↓
Application says:
"Job completed"
     |
     ↓
QueueLess
```

---

### 4. Request lifecycle ≠ Job lifecycle

- #### Problem when the job lifecycle is the part of request lifecycle.
```Flow
Tomcat Thread
    │
    ├── calls QueueLess
    │
    │   WAITING...
    │
    │   WAITING...
    │
    ↓
QueueLess response
    │
    ↓
Tomcat Thread continues
```

- That Tomcat thread is occupied while waiting for the synchronous HTTP response.

- #### Hence we don' t  want the consumer application's worker thread to wait until the queueless sends JOB_READY response back

- the flow may look like this :
```Flow
POST /videos
       ↓
Consumer Application
       ↓
QueueLess
       ↓
"Job accepted: 123"
       ↓
Consumer Application
       ↓
HTTP 202 Accepted
       ↓
Client

Tomcat Thread -(Consumer application tomcat thread)
    │
    ├── receive request
    ├── submit job
    ├── receive acknowledgement
    └── return 202
             ↓
          THREAD FREE
          

Later:

QueueLess
    │
    │ JOB_READY
    ↓
Consumer Application
    │
    ↓
Process Job

```


---
##  How does the consumer submit work?

 - The consuming application might have a controller.

```
POST /some-business-operation
        ↓
Consumer Controller
        ↓
QueueLess Client
        ↓
QueueLess
```

- application submits job to queueless.

- QueueLess responds something like:
```Note
202 Accepted

jobId = 123
status = QUEUED
```

- The consuming application can tell its user:
```
Your request has been accepted.

Queue position: 7
Estimated wait: 14 minutes
```


---

## 6. How does QueueLess tell the application that the job is ready?

### Option A — HTTP callback

The consuming application gives QueueLess a callback endpoint.
```
Application A

callback:
https://app-a.com/api/queueless/job-ready
```

- so when the job is ready:
```
QueueLess
    |
    | HTTP POST
    ↓
Application A's callback endpoint
```

- The application then processes the job.

### Option B — Message broker
- APACHE KAFKA
```
QueueLess
    |
    | JOB_READY
    ↓
Kafka
    |
    ↓
Consumer Application
```

- The application consumes the event and starts processing the job.


The important architectural principle is:

> **The notification that a job is ready should be asynchronous.**


---

## Two Controllers Idea

- Controller 1 - User submits Work.
- Controller 2- QueueLess tells application that the work is ready and should be proccessed.
- This separates the **initial request** from the **later job execution**.

---

## Callback URL cannot be hard-coded
- Different Applications will have different end points.
- Different Queues will again have different endpoints.
- hence we don't hardcode the call back endpoints. 
- the consuming application needs to **register/configure its callback destination**.


- When Creating a Queue:
```
{
  "queueName": "video-processing",
  "callback": {
    "url": "https://video-service/api/queueless/job-ready"
  }
}
```

```
Queue
 ├── queueId
 ├── name
 ├── callbackUrl
 ├── schedulingStrategy
 └── configuration
```

---
- it has a similarity in integration pattern with the OAuth.
The similarity is:

> **External applications register/configure themselves with an independent service and then interact with that service through a defined contract.**

---
##  QueueLess must know something about each job

- Hence the onsuming application needs to provide **scheduling metadata**.
```
{
  "jobId": "job-123",
  "priority": 5,
  "estimatedDuration": 120,
  "deadline": "2026-08-19T18:00:00",
  "createdAt": "2026-08-19T17:10:00"
}
```

---
## Queue configuration vs Job metadata

 - Queue-Level Configuration
```
{
  "queueName": "video-processing",
  "schedulingStrategy": "ADAPTIVE",
  "maxConcurrentJobs": 10
}
```

- Job-level information
```
{
  "jobId": "123",
  "priority": 7,
  "estimatedDuration": 180,
  "deadline": "...",
  "createdAt": "..."
}
```

```
QUEUE
 ├── scheduling strategy
 ├── concurrency
 ├── priority rules
 └── configuration

JOB
 ├── ID
 ├── priority
 ├── estimated duration
 ├── creation time
 ├── deadline
 ├── status
 └── metadata
```

- The important point is QueueLess should'nt understand the business-specific data.
- It primarily needs to understand the scheduling metadata.

---
## Not all metadata has to be mandatory

- what if the application 's particular request's estimatedDuration is'nt known??
  ```
estimated duration provided?
          |
      ┌───┴───┐
     YES      NO
      |        |
      ↓        ↓
    use      predict
             |
       historical data
  ```
  - If there is no Historical Data either, QueueLess can use a default estimate.

---
## Adaptive FeedBack Loop

```
Job submitted
      ↓
Historical data
      ↓
Estimated duration
      ↓
Scheduling decision
      ↓
Actual execution
      ↓
Actual duration
      ↓
Historical data updated
      ↓
Future scheduling improves
```

---
### Adaptive Scheduling engine

Possible strategies:
```
FIFO
Priority
Shortest Estimated Job
Deadline-aware
Adaptive
```

---
## 17. QueueLess should react to real-time events

When an event occurs:
```
Event
  ↓
Queue state changes
  ↓
Scheduler reevaluates
  ↓
New schedule
  ↓
Updated positions/estimates
```

```
JOB_SUBMITTED
JOB_CANCELLED
JOB_STARTED
JOB_COMPLETED

RESOURCE_AVAILABLE
RESOURCE_UNAVAILABLE

PRIORITY_CHANGED
```
![[Pasted image 20260819191707.png]]

---
## Resources

![[Pasted image 20260819192136.png]]

----

## Redis, KAFKA ,Postgre SQL integrations
![[Pasted image 20260819191707.png]]

```
Kafka
→ transports events

Redis
→ fast current state

PostgreSQL
→ persistent state

QueueLess
-> can use Kafka, Redis, PostgreSql
→ orchestration + scheduling + adaptation
```

