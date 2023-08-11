# Encrypt EBS snapshot

## Behavior
1. The script reads the snapshot IDs from the snap_list.txt file and starts encrypting the first 20 snapshots without waiting for their status.
2. As encrypted snapshots are completed, new snapshots are started in their place, maintaining a maximum of 20 in-progress snapshots.
3. The script ensures that snapshots are not duplicated during the encryption process.
4. The process continues until all snapshot IDs mentioned in the text file are successfully encrypted.

### To run the code we will require boto3 client and Python, also AWS CLI.


####  1. Install AWS CLI and Configure 

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws configure
```

#### 2. Install pythonand boto3

```
sudo apt install -y python3-pip
```
#### 3. Create a requirements.txt file and add the following dependency

```
vim requirements.txt
boto3
## After added the denpendency and install the dependency
pip install -r requirements.txt
```
#### 4. Setup the environment variables as mentioned below.

```
export AWS_ACCESS_KEY= <Your access key id here>
export AWS_SECRET_KEY= <Your secret key id here>
export AWS_REGION=<The region to work on>
export AWS_ACCOUNT_ID=<Your AWS AccountID>
export KMS_KEY_ID= <KMS KEY ID of /aws/ebs>  ## this key id can be found on aws console
```
![image](https://github.com/sagarshrestha24/myproject/assets/76894861/4a99dcbf-d6a9-453d-bf1a-71eb5457145b)

#### 5. Create a TEXT file named snap_list.txt where the snapshot ids will be placed extracting from the AWS.

#### 6. Run the python code.

```
python <yourfilename.py>
```
Optimized Code

```
import boto3
import time
import os
def encrypt_snapshot(snapshot_id, kms_key_id, aws_account_id, aws_region):
    client = boto3.client(
        'ec2',
        aws_access_key_id= os.environ['AWS_ACCESS_KEY'],
        aws_secret_access_key= os.environ['AWS_SECRET_KEY'],
        region_name=aws_region
    )

    copy_snapshot_args = {
        'SourceRegion': aws_region,
        'SourceSnapshotId': snapshot_id,
        'Encrypted': True,
        'KmsKeyId': 'alias/aws/ebs',
        'Description': "Created from "+snapshot_id
    }

    response = client.copy_snapshot(**copy_snapshot_args)
    return response['SnapshotId']

def check_snapshot_status(snapshot_id, aws_region):
    client = boto3.client(
        'ec2',
        aws_access_key_id=os.environ['AWS_ACCESS_KEY'],
        aws_secret_access_key=os.environ['AWS_SECRET_KEY'],
        region_name=aws_region
    )
 
    response = client.describe_snapshots(SnapshotIds=[snapshot_id])
    status = response['Snapshots'][0]['State']
    return status

# Read snapshot IDs from file
snapshot_file = 'snap_list.txt'
kms_key_id = os.environ.get('KMS_KEY_ID', '')
aws_account_id = os.environ['AWS_ACCOUNT_ID']
aws_region = os.environ['AWS_REGION']

with open(snapshot_file, 'r') as file:
    snapshot_ids = file.read().splitlines()



snapshot_index = 0
max_concurrent_encryptions = 20
concurrent_encryptions = 0
encrypted_snapshots = []
 
while snapshot_index < len(snapshot_ids) or concurrent_encryptions > 0:
    if snapshot_index < len(snapshot_ids) and concurrent_encryptions < max_concurrent_encryptions:
        snapshot_id = snapshot_ids[snapshot_index]
        encrypted_snapshot_id = encrypt_snapshot(snapshot_id, kms_key_id, aws_account_id, aws_region)
        encrypted_snapshots.append(encrypted_snapshot_id)
        print(f"Encrypted snapshot {encrypted_snapshot_id} created.")
        snapshot_index += 1
        concurrent_encryptions += 1
 
    completed_snapshots = []
    for encrypted_snapshot_id in encrypted_snapshots:
        status = check_snapshot_status(encrypted_snapshot_id, aws_region)
        if status == 'completed':
            print(f"Encrypted snapshot {encrypted_snapshot_id} completed.")
            concurrent_encryptions -= 1
            completed_snapshots.append(encrypted_snapshot_id)
 
    encrypted_snapshots = [s for s in encrypted_snapshots if s not in completed_snapshots]
    time.sleep(5)  # Adjust the delay time as needed
 
print("All snapshots have been successfully encrypted.")

```
