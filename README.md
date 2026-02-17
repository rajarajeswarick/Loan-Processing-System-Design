# Loan-Processing-System-Design
Synchronous and asynchronous Loan Processing System

User submits a loan application
System gives an immediate acknowledgment
Backend performs credit check + fraud detection in parallel
System ensures no message loss and no duplicate processing
Final result is written once and can trigger notifications

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/2788360d-831c-4abc-8a9d-32a878672034" />

1) User submits the loan application

User → API Gateway
The client (web/mobile) calls an API like:
POST /loan-applications
API Gateway provides:Authentication/authorization (Cognito/JWT/IAM)
Throttling / rate limiting
WAF protection (optional)
Request validation

2) Fast acknowledgment (Synchronous part)

API Gateway → “SubmitLoan” Lambda → immediate response
Inside the submit Lambda, we do only fast work:

2.1 Generate an applicationId
Create a unique ID for the loan application.
Example: UUID or a business key.

2.2 Idempotency control (no duplicates)
This is critical.
If the client retries the request (network glitch, double click), you must not create 2 applications.
How:Store applicationId (or a requestId) in DynamoDB using a conditional write:
“Insert only if applicationId does not exist”

2.3 Write initial record to DynamoDB
Save:applicationId
customer data (or S3 pointer)
status = RECEIVED
createdAt timestamp

2.4 Return 202 Accepted
API Gateway returns to user immediately:
202 Accepted
applicationId
maybe “check status endpoint”:
GET /loan-applications/{applicationId}
This satisfies: “User must get immediate acknowledgment.”

3) Kick off background workflow (Asynchronous part)
After the 202 is returned, backend continues.
Option shown in diagram:
API Gateway → SNS (optional event notification)
SNS can broadcast the “ApplicationReceived” event to multiple subscribers.
In many real builds, Step Functions is started directly from Lambda.
SNS is useful if you also want:
audit subscriber
analytics subscriber
notification subscriber

4) Step Functions orchestration begins
Start Step Functions execution
Step Functions becomes your “conductor”
It manages: parallel processing
retries/timeouts
workflow state
failure paths

5) Parallel State: Credit check + Fraud detection
Step Functions runs two branches at the same time.
Branch A: Credit Check (heavy compute)
Credit Check → ECS/Fargate or EC2 (or Lambda if light)
Because credit check can be heavier / longer-running, you might run it on:
ECS Fargate (best scale + managed)
EC2 workers (if custom legacy libs)
Lambda (if within runtime/CPU limits)
Typical pattern (production):
Step Functions → SQS → ECS/EC2 worker consumes → returns result
Output example:
creditScore
creditDecision = PASS/FAIL
reason codes

Branch B: Fraud Detection (serverless)
Fraud Detection → Lambda
Lambda evaluates fraud rules/models.
Output:
fraudRiskScore
fraudDecision = PASS/REVIEW/FAIL

6) Join results (merge)
When both branches finish, Step Functions merges results:
credit result
fraud result
Now Step Functions has enough info to decide:
Approved
Rejected
Needs manual review

7) Single Writer Update to DynamoDB (prevents duplication)
This is how you avoid the duplication problem you pointed out.
Only Step Functions (or one final Lambda) writes final decision.
It updates the application record:
status = APPROVED / REJECTED / REVIEW
stores credit + fraud results
updatedAt
This avoids race conditions like:
credit writes APPROVED while fraud writes FAIL at the same time.

8) Retries + failures + DLQ

8.1 Retries
Step Functions can retry failed steps with backoff.
Example:
retry credit check 2–3 times
retry fraud Lambda on transient errors

8.2 DLQ handling (shown)
If you use SQS between workflow and workers:
failures cause message to be retried
after maxReceiveCount, message goes to DLQ
DLQ is your “parking lot for poison messages”

8.3 CloudWatch alarms
CloudWatch monitors:
DLQ depth > 0 (alarm)
Step Functions failed executions (alarm)
Lambda error rate (alarm)
age of oldest message in queue (alarm)
So ops team knows immediately.

9) How user gets final status
Two common patterns:
Option A: Polling (simple)
Client calls:
GET /loan-applications/{applicationId}
API reads DynamoDB and returns current status.

Option B: Callback / Notification (nice UX)
When final decision is ready:
SNS / SES sends email/text
WebSocket / AppSync / EventBridge pushes update
