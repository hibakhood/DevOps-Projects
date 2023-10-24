### Prerequisites 

- An AWS account and user with admin permissions
- Install AWS CLI
- Install python 3.x on your local PC.

## Implementation Steps

#### Step 1: Initiate a Glacier retrieval job and download it's job to a file.

1. Start an inventory retrieval job and ensure to save the json output of the command which contains a job ID.
    ```
    aws glacier initiate-job --vault-name <vault_name> --account-id <account_id> --job-parameters '{"Type": "inventory-retrieval", "Description": "Job to get inventory"}' | cat
    ```

2. Use the command below to check the status of the retrieval job. Wait for the job to complete, this can take up to 5hrs. Once the job completes, the "StatusCode" value will change from "Inprogress" to "Succeeded".
    ```
    aws glacier describe-job --vault-name <vault_name> --account-id <account_id> --job-id <jobID> | cat
    ```

3. Download the retrieval job inventory to a file named output.json. It will contain the ids of the archives of the vault.
    ```
    aws glacier get-job-output --vault-name <vault_name> --account-id <account_id> --job-id <jobID> output.json
    ```

#### Step 2: Split the ids of archives into several files

- Use the generate_split_files.py script in this repo and the output.json from Step 1.3 to split the vault's archive ids into several files
    - Run the command below to extract the archive ids from output.json file and split them into several files.
        ```
        python generate_split_files.py -i <json_file> -n <num_split_files>
        ```

The script will create a directory named archives and generate a number of txt files (named filex.txt) containing the ids of the archives. This page shows an example of the script executed and expected output. 

#### Step 3: Create a stack in CloudFormation to create S3 bucket
- Create a bucket by deploying this CloudFormation template bucket.yaml, to the same region as the vault.
    - It will create a bucket with this naming convention "glacier-archive-remover-<account_number>-<region>"

#### Step 4: Upload the lambda code and txt files to S3  
- Upload the files below to the root of the S3 bucket created in the previous step "glacier-archive-remover-<account_number>-<region>" 
    - Upload the "archives" folder which includes txt files generated by the script from step 2
    - Upload the s3_bucket_reader.zip file 
    - Upload the delete_glacier_archive.zip file 

#### Step 5: Create a stack in CloudFormation to create a SQS Queue, Lambda function, Log Group and IAM Role
- Deploy the "main_template.yaml" to create the required resources for this solution

    - Fill in the name of the vault to be deleted under the "Glacier Vault Name" parameter value (this is REQUIRED). 
    - Accept the remaining default parameters 

#### Step 6: Invoke lambda function named S3BucketReader
aws lambda invoke --function-name S3BucketReader response.json

#### Step 7: Verify archives are being deleted
Note that depending on the number of archives, deleting all archives can take up to a day or more. To confirm archives are being deleted. There are two options:

- Check that the files in the bucket are being deleted or reduced from their original file size OR
- The Cloudwatch function's logs can be reviewed for the log below.
    ```
    All archives listed in archives/filex.txt were successfully deleted
    ```

#### Step 8: Deleting the Empty Vault
Once there are no more files in S3 delete the vault using the AWS CLI command below. Note that there can be a significant lag (up to hours) between when the S3 files are deleted and when the vault is emptied. Another inventory might need to be taken to verify if the vault has been emptied using the commands in steps 1.2. and 1.3.
    ```
    aws glacier delete-vault --vault-name <vault_name> --account-id <account_id>
    ```

#### Step 9: Clean up Change resources
   - Empty S3 bucket with name format "glacier-archive-remover-<account_number>-<region>"
   - Delete all CloudFormation stacks 