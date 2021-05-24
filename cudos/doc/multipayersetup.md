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


## Configuring the source accounts roles and buckets


For each of the CUR buckets in each of the payer accounts:
* Configure versioning on the bucket (needed for replication)
* Optional: Add a lifecycle policy to remove old versions of objects after 7 days: Under Management, Lifecycle rules section click on create lifecycle rule. Add a lifecycle rule like this: 
 
  ![Alt text](/cudos/doc/resources/lifecycledeleteoldversions.png?raw=true "Lifecycle Delete Old Versions")
* open the IAM console in a new browser tab, create a new role that with a trusted entity type is AWS Servive, the service, S3 . 
  * If you decide that the consolidated buckets is in one of the payers then the role for that payer account where the consolidated bucket should look like the code sample below. Remember to repace the resources *cur-cudos-111* with your source payer1 bucket(in 2 places below), *cudos-cur-combined-111* with your destination CUR bucket(in one place).
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetReplicationConfiguration",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::cur-cudos-111"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObjectVersionForReplication",
                "s3:GetObjectVersionAcl",
                "s3:GetObjectVersionTagging"
            ],
            "Resource": [
                "arn:aws:s3:::cur-cudos-111/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ReplicateObject",
                "s3:ReplicateDelete",
                "s3:ReplicateTags"
            ],
            "Resource": "arn:aws:s3:::cudos-cur-combined-111/*"
        }
    ]
  }
    ```

  * for payer accounts that would need to do cross account replication the role should look like this. Remember to replace your source and bucket names accordingly, in this situation *cur-cudos-222* needs to be replaced with source bucket name in 2 places and *cudos-cur-combined-111* with the destination bucket :
   ```json
     {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetReplicationConfiguration",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::cur-cudos-222"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObjectVersionForReplication",
                "s3:GetObjectVersionAcl",
                "s3:GetObjectVersionTagging"
            ],
            "Resource": [
                "arn:aws:s3:::cur-cudos-222/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ReplicateObject",
                "s3:ReplicateDelete",
                "s3:ReplicateTags",
                "s3:ObjectOwnerOverrideToBucketOwner"
            ],
            "Resource": "arn:aws:s3:::cudos-cur-combined-111/*"
        }
    ]
    }
   ```
* Setup bucket replication: Under management replication rules:
  * Create a new replication rule, similar to the following:
    ![Alt text](/cudos/doc/resources/replication_rule_common.png?raw=true "Lifecycle Delete Old Versions")
  * Destination settings would be different if one of the payer accounts is also hosting the consolidated bucket, below is an example for same account replication.
    ![Alt text](/cudos/doc/resources/replication_rule_same_payer.png?raw=true "Lifecycle Delete Old Versions")
  * for cross account replication it should look like this
    ![Alt text](/cudos/doc/resources/replication_rule_cross_account.png?raw=true "Lifecycle Delete Old Versions")

* Test the replication by uploading a different key test object into each of the source buckets and make sure all objects show in the destination bucket, also make sure you can download and view objects.

* Now that replication is setup for all buckets we need a one time sync of existing data: 

  * Start with the payer account that is in the same account as the destination bucket if you have one (if you have other prefixes in the CUR bucket use exclude/include) patterns as below (assumes *payer1* aws cli profile is setup with an access key that has access to both buckets in this account). Remember to replace accordingly your bucket names and prefixes: 
    ```
    aws s3 sync s3://cudos-cur-111 s3://cudos-cur-combined-111 --exclude "*" --include "payer1-reportname/payer-1-cur-prefix/*"  --profile payer1
    ```
  * For any cross account sync, use a cli profile that has full s3 rights in the source account, the command is similar but needs  **bucket owner full control acl ** otherwise it will not work. Remember to replace accordingly your bucket names and prefixes:
    ```
    aws s3 sync s3://cudos-cur-222 s3://cudos-cur-combined-111 --exclude "*" --include "payer2-reportname/payer2-prefix/*"  --acl bucket-owner-full-control --profile payer2

    ```

















