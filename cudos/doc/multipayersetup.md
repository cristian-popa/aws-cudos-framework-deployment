# Multi Payer Setup

## Scenario

Customer with multiple payer accouns wants to consolidate the CUDOS Dashboards under a single account.
In this example:</br>
&nbsp;&nbsp; Account 111111111111 is *payer1* with cur bucket *cur-cudos-111*</br> 
&nbsp;&nbsp; Account 222222222222 is *payer2* with cur bucket *cur-cudos-222*<br>

We will centralize a cur bucket in payer 1 account and will setup athena integration and quicksight cudos dashboards in this account. You could designate a non payer account for consolidation if you wish to do so with minimal modifications. If you decide to replicate into a separate cost governance account you simply skip the same account bucket replication setup for that payer and use the cross account replication instructions. 



> :warning: This guide works for objects encrypted with either SSE-S3 or not encrypted objects. For SSE-KMS encrypted objects additional policy statements and replication configuration will be needed:  see https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-config-for-kms-objects.html 


## Prerequisites

1. Prepare client https://cudos.workshop.aws/prerequisites/prepare-aws-cli.html
2. For each payer account setup CUR per instructions here: https://cudos.workshop.aws/prerequisites/prepare-cur.html.</br> 
>:warning: Provide a different name for the CUR report for each of the buckets so that we will have differen prefixes. *cur-111* and *cur-222* (for example)
3. Designate the account that you want to host the consolidated S3 CUR data for both payers in our case *payer1* account


## Creating the consolidated bucket

In your designated account where you want to consolidate all CUR reports, create a bucket, for this example we will use *cudos-cur-combined-111*

* Make sure versioning is enabled (required for replication)
* Add the following bucket policy 
    * make sure to replace the Principal.AWS array with the correct list of payer accounts that would replicate to this bucket) ] . In this example replace *["222222222222","111111111111"]* with the list of your payer accounts in 3 sections of the below policies . [TODO - check if you can also add the payer account that you use to host the consolidated bucket as well, I have only tested with the other bucket, it shoulw work then delete this comment]
    * replace the bucket name according to the bucket name you chose for your bucket. In this example replace *cudos-cur-combined-111* with your consolidated bucket name.

    ```json
  {
    "Version": "2008-10-17",
    "Id": "PolicyForCombinedBucket",
    "Statement": [
        {
            "Sid": "Set permissions for objects",
            "Effect": "Allow",
            "Principal": {
                "AWS": ["222222222222","111111111111"]
            },
            "Action": [
                "s3:ReplicateObject",
                "s3:ReplicateDelete"
            ],
            "Resource": "arn:aws:s3:::cudos-cur-combined-111/*"
        },
        {
            "Sid": "Set permissions on bucket",
            "Effect": "Allow",
            "Principal": {
                "AWS": ["222222222222","111111111111"]
            },
            "Action": [
                "s3:List*",
                "s3:GetBucketVersioning",
                "s3:PutBucketVersioning"
            ],
            "Resource": "arn:aws:s3:::cudos-cur-combined-111"
        },
        {
            "Sid": "Set permissions to pass object ownership",
            "Effect": "Allow",
            "Principal": {
                "AWS": ["222222222222","111111111111"]
            },
            "Action": [
                "s3:ReplicateObject",
                "s3:ReplicateDelete",
                "s3:ObjectOwnerOverrideToBucketOwner",
                "s3:ReplicateTags",
                "s3:GetObjectVersionTagging",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::cudos-cur-combined-111/*"
        }
    ]
  }
  ```



* Under Permissions, Object Ownership also make sure to check Bucket owner preferred [TODO: you could add a screenshot, under resources folder and reference it here]


## Configuring the source acconts and buckets

For each of the CUR buckets in each of the payer accounts:
* Configure versioning on the bucket (needed for replication)
* Optional: Add a lifecycle policy to remove old versions of objects after 7 days: Under Management, Lifecycle rules section click on create lifecycle rule. Add a lifecycle rule like this: 
![Alt text](/cudos/doc/resources/lifecycledeleteoldversions.png?raw=true "Lifecycle Delete Old Versions")











