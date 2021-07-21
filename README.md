# batch_job_log
Logging AWS Batch job data upon finish

# Logging in AWS Batch
As it is documented in [logging section](https://docs.aws.amazon.com/batch/latest/userguide/using_awslogs.html) of AWS Batch, the default place for each job is:

```/aws/batch/job/<job-definition-name>/default/<ecs-task-id>```

Since Batch uses ECS as a backend service, this log path has the ECS task ID in the log-stream name, which is different from Batch job ID. The only way to make a connection between these two is the [DescribeJobs](https://docs.aws.amazon.com/batch/latest/APIReference/API_DescribeJobs.html) API call, which will return the CloudWatch log group and log stram name based on the Batch job ID we provide.

Even though you have the job ID right after the [SubmitJob](https://docs.aws.amazon.com/batch/latest/APIReference/API_SubmitJob.html) API call as a return value, you cannot use this to find the log stream name right ahead, since it will only be determined once the job transforms from SUBMITTED state into RUNNING (with PENDING and/or RUNNABLE in between) and could only be retrieved for another 24 hours after the job have finished (by either SUCCEEDED or FAILED state). 

Hence, it may happen to miss the opportunity to set up a relationship between your job IDs and the corresponding task IDs (hence log stream names). 

In order to mitigate this issue, you can create a [CloudWatch event](https://docs.aws.amazon.com/batch/latest/userguide/batch_cwet.html) on batch job state change events, triggering a lambda function which will grab the required information from this event and store it into a DynamoDB table. This give you a clean, serverless solution for this problem.

In order to demonstrate further possibilities, the Lambda function also uses [DescribeContainerInstances API](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_DescribeContainerInstances.html) to retrieve additional information about the instance catered the job. However, DescribeContainerInstances requires the container instance ID as well as the name of the cluster it resides. The cluster name is only available from the previous event if your container instance is already using the [new ARN format](https://docs.aws.amazon.com/AmazonECS/latest/userguide/ecs-account-settings.html#ecs-resource-ids), since the old arn format is:
```arn:aws:ecs:region:aws_account_id:container-instance/container-instance-id```
while the new ARN format:
```arn:aws:ecs:region:aws_account_id:container-instance/cluster-name/container-instance-id```

Consequently, in case you want to describe the container instances to retrieve additional information, you need to [opt in to use the new ARN format for ECS](https://docs.aws.amazon.com/AmazonECS/latest/userguide/ecs-modifying-longer-id-settings.html) if you haven't already done so.

Using the CloudFormation template in this repository, you can set up the above infrastructure in a specific region/account, creating the following resources:

- A DynamoDB table with the relevant information about ech AWS Batch job finished. The name of this table is output of the stack
- A CloudWatch event firing on batch job state change events where the even t is "SUCCEEDED" or "FAILED" (e.g. finished)
- A Lambda function called by the CloudWatch event above, processing the event passed to it and storing useful data into the DynamoDB table
- A role with two policies used by lambda in order to be able to run and write the DynamoDB table
- Lambda execution role for CloudWatch to be able to execute the lambda

By the end of the stack creation, all you need is the DynamoDB table name which is output of the stack.

![PIC](https://raw.githubusercontent.com/konjanos/batch_job_log/main/batch_log.svg)
