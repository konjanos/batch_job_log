# batch_job_log
Logging AWS Batch job log stream names upon finish

# Logging in AWS Batch
As it is documented in [logging section](https://docs.aws.amazon.com/batch/latest/userguide/using_awslogs.html) of AWS Batch, the default place for each job is:

```/aws/batch/job/<job-definition-name>/default/<ecs-task-id>```

