# VPC Flow Logs

While there are a few different ways to get VPC Flow Logs into Splunk in a Multi-Account environment, Cenovus requested to achieve this via a Kinesis stream; this decision came after consulting with Splunk Professional Services as well as AWS Professional Services.

In order to achieve this setup, we needed to use CloudWatch to forward data cross-account.

The centralization of the VPC Flow Logs was done into the log-archive account.

## Kinesis Stream

A Kinesis stream was created with the following details:

```
Region:
us-west-2

Stream name:
VPCFlowLogsRecipientStream

Shards:
10

log-archive account ID:
191161022481

Kinesis stream ARN:
arn:aws:kinesis:us-west-2:<log-archive-account-id>:stream/VPCFlowLogsRecipientStream
```

## CloudWatch Destination

A CloudWatch Logs destination was created in the log-archive account to receive all VPC Flow Logs from each AWS Account.

Note: This cannot be done via the Web Console, but it can be done via the AWS CLI.

```
aws logs put-destination \
    --destination-name "VPCFlowLogsCentralizedDestination" \
    --target-arn "arn:aws:kinesis:us-west-2:<log-archive-account-id>:stream/VPCFlowLogsRecipientStream" \
    --role-arn "arn:aws:iam::<log-archive-account-id>:role/Cenovus-CWL-to-Kinesis-for-FlowLogs-to-Splunk-role"
```

CloudWatch Logs destination ARN:

```
arn:aws:logs:us-west-2:<log-archive-account-id>:destination:VPCFlowLogsCentralizedDestination
```

The CloudWatch Logs destination needs to have a policy attached to control access to it. The following access policy is listing the AWS Accounts that are allowed to send data to this CloudWatch destination.

Note: At the moment of the implementation of this setup the CloudWatch destination access policy did not support using the AWS Organizations ID and required an explicit list of each AWS account ID. This requires future updates of this policy every time a new AWS Account is added or is terminated.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "<member-account-0>",
          "<member-account-1>",
          "<member-account-2>",
          "<member-account-3>",
          "<member-account-5>"
        ]
      },
      "Action": "logs:PutSubscriptionFilter",
      "Resource": "arn:aws:logs:us-west-2:<log-archive-account-id>:destination:VPCFlowLogsCentralizedDestination"
    }
  ]
}
```

Attaching the policy to the destination is also a CLI command (assuming the above policy is written in the destination-access-policy.json file):

```
aws logs put-destination-policy \
    --destination-name "VPCFlowLogsCentralizedDestination" \
    --access-policy file://~/destination-access-policy.json
```

## IAM Role for CloudWatch

This is a dedicated IAM Role that has been created for the VPC Flow Logs integration. This IAM Role gives CloudWatch Logs permissions to put data into the Kinesis stream.

```
IAM Role name:
Cenovus-CWL-to-Kinesis-for-FlowLogs-to-Splunk-role

ARN:
arn:aws:iam::<log-archive-account-id>:role/Cenovus-CWL-to-Kinesis-for-FlowLogs-to-Splunk-role
```

### Trust Policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "logs.us-west-2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### IAM Policy (in line)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "kinesis:PutRecord",
      "Resource": "arn:aws:kinesis:us-west-2:<log-archive-account-id>:stream/VPCFlowLogsRecipientStream",
      "Effect": "Allow"
    },
    {
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": "cloudwatch.amazonaws.com"
        }
      },
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::<log-archive-account-id>:role/Cenovus-CWL-to-Kinesis-for-FlowLogs-to-Splunk-role",
      "Effect": "Allow"
    }
  ]
}
```

## CloudWatch Log Group

To leverage the setup implemented in the log-archive account (described above), each AWS Account that has a VPC that needs monitoring, will have to have a dedicated CloudWatch Log Group created.

```
Name:
VPCFlowLogs

ARN:
arn:aws:logs:us-west-2:<member-account-id>:log-group:VPCFlowLogs:*
```

### Subscription:

```
aws logs put-subscription-filter \
    --log-group-name "VPCFlowLogs" \
    --filter-name "AllVPCFlowLogs" \
    --filter-pattern "" \
    --destination-arn "arn:aws:logs:us-west-2:<log-archive-account-id>:destination:VPCFlowLogsCentralizedDestination"
```

If this is successful, the subscription can be seen in the CloudWatch Log Group.

Note: the subscription filter above is the method that allows pushing data cross-account; as it can be seen, the destination-arn is pointing to the CloudWatch Logs destination created in the log-archive account.

## Flow Logs

For each VPC of interest, in each AWS Account, VPC Flow Logs were enabled and configured as follows:

```
Filter:
All

Maximum aggregation interval:
10 min

Destination:
Send to CloudWatch Logs

Destination log group:
VPCFlowLogs

IAM Role:
arn:aws:iam::<member-account-id>:role/Cenovus-publish-to-CWL-role
```

### Trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "vpc-flow-logs.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### IAM Policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup", "logs:DescribeLogGroups"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogStream", "logs:DescribeLogStreams"],
      "Resource": "arn:aws:logs:us-west-2:[member-account-id]:log-group:VPCFlowLogs:*"
    },
    {
      "Effect": "Allow",
      "Action": "logs:PutLogEvents",
      "Resource": "arn:aws:logs:us-west-2:[member-account-id]:log-group:VPCFlowLogs:log-stream:*"
    }
  ]
}
```
