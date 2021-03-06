DDBImportExport is a command line tool to import data from JSON files into DynamoDB table, or export data from DynamoDB table into JSON files. It also provides a data generation utility for quick testing.

## Installation

DDBImportExport requires Python 3, Virtual Environments (venv), and boto3.

On a newly launched EC2 instance with Amazon Linux 2, install DDBImportExport with the following commands:

~~~~
sudo yum install git python3 -y
python3 -m venv boto3
source boto3/bin/activate
pip install pip --upgrade
pip install boto3
git clone https://github.com/qyjohn/DDBImportExport
~~~~

On a newly launched EC2 instance with Ubuntu 18.04, install DDBImportExport with the following commands:

~~~~
sudo apt update
sudo apt install python3 python3-venv git -y
python3 -m venv boto3
source boto3/bin/activate
pip install pip --upgrade
pip install boto3
git clone https://github.com/qyjohn/DDBImportExport
~~~~

You need the following IAM permissions to use DDBImportExport:

- dynamodb:DescribeTable
- dynamodb:Scan
- dynamodb:BatchWriteItem
- s3:GetBucketLocation
- s3:ListBucket
- s3:ListObjects
- s3:GetObject
- s3:PutObject

Below is an example that works:

~~~~
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:ListObjects",
                "s3:GetBucketLocation"
            ],
            "Resource": [
                "arn:aws:s3:::bucket-name",
                "arn:aws:s3:::bucket-name/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:DescribeTable",
                "dynamodb:Scan",
                "dynamodb:BatchWriteItem",
                "dynamodb:PutItem",
                "dynamodb:GetItem"
            ],
            "Resource": "*"
        }
    ]
}
~~~~

## DDBImport

DDBImport is a python script to import from JSON file into DynamoDB table. The following parameters are required:

| parameter  |  description |
|---|---|
| -r | The name of the AWS region, such as us-east-1. |
| -t | The name of the DynamoDB table. |
| -s | The input source. Supports local file, local folder, S3 object and S3 prefix.  |
| -p | The number of sub-processes (threads) to use. |
| -c | The maximum amount of write capacity units (WCU) to use. |

Usage:

~~~~
python DDBImport.py -r <region> -t <table> -s <source> -p <processes> -c <capacity>
~~~~

Example:

~~~~
python DDBImport.py -r us-east-1 -t TestTable -p 8 -c 1000 -s /home/ec2-user/test.json
python DDBImport.py -r us-east-1 -t TestTable -p 8 -c 2000 -s /home/ec2-user/data/ 
python DDBImport.py -r us-east-1 -t TestTable -p 8 -c 1000 -s s3://bucket-name/prefix/test.json 
python DDBImport.py -r us-east-1 -t TestTable -p 8 -c 2000 -s s3://buckeet-name/data/ 
~~~~
  
The script launches multiple processes to do the work. The processes poll from a common queue for data to write. When the input source contains only a single file (or S3 object), the queue contains the file content. When the input source contains more than a single file (or S3 objects), the queue contains the file names (or S3 object names). Obviously, the number of processes needs to be smaller than the number of files or S3 objects.

It is recommended that you use either a fixed provisioned WCU or an on-demand table for the import. The import creates a short burst traffic, which is not friendly for the DynamoDB auto scaling feature. The BatchWriteItem API is used to perform the import. Assuming that each item is less than 1 KB, then each item consumes 1 WCU. When you have bigger items, the consumed WCU can be higher.

It is safe to use 1 process per vCPU core. If you have an EC2 instance with 4 vCPU cores, it is OK to set the process count to 4. Each process can handle approximately 1000 items in a second. The consumed WCU depends on the size of the items. Assuming that each item is less than 1 KB, then each process requires approximately 1000 WCU. If you use 8 processes to do the import, you need 8000 provisioned WCU on the table. When you have bigger items, the consumed WCU can be higher for each process. It is recommend that you use up to 1000 WCU per process. For example, if you use 8 processes for the import (-p 8), you should use less than 8000 WCU (-c 8000).

Tested on an EC2 instance with the c3.8xlarge instance type. The data set contains 10,000,000 items, with each item being approximately 170 bytes. The size of the JSON file is 1.7 GB. The DynamoDB table has 40,000 provisioned WCU. Perform the import with 32 threads, and the import is completed in 7 minutes. The peak consumed WCU is approximately 32,000 (average value over a 1-minute period).

## DDBExport

DDBExport is a python script to export data from DynamoDB table into JSON file. The following parameters are required:

| parameter  |  description |
|---|---|
| -r | The name of the AWS region, such as us-east-1. |
| -t | The name of the DynamoDB table. |
| -p | The number of sub-processes (threads) to use. |
| -c | The maximum amount of read capacity units (RCU) to use. |
| -s | The maximum size of each individual output file, in MB. |
| -d | The output destination. Supports both local folder and S3. |

Usage:

~~~~
python DDBExport.py -r <region> -t <table> -p <processes> -c <capacity> -s <size> -d <destination>
~~~~

Example:

~~~~
python DDBExport.py -r us-east-1 -t TestTable1 -p 8 -c 1000 -s 1024 -d /data
python DDBExport.py -r us-west-2 -t TestTable2 -p 8 -c 2000 -s 2048 -d s3://bucket/prefix/
~~~~

With a small table (at GB scale), it is safe to use 1 process per vCPU core. If you have an EC2 instance with 4 vCPU cores, it is OK to set the process count to 4. However, it is important that you have sufficient provisioined RCU on the table, and specify sufficient max capacity for the export with the -c option. 

Depending on the number of sub-processes you use and the maximum size of the output file, DDBExport will create multiple JSON files in the output destination. The name of the JSON files will be TableName-WorkerID-FileNumber.json. 

It is important that you have sufficient free space in the output destination. When using S3 as the output destination, your current folder needs to have sufficient free space to hold the intermediate data, which is the number of sub-processes times the size of each individual output file. For example, if the number of sub-processes is 8, and the size of each individual output file is 1024 MB, then you will need 8 x 1024 MB = 8 GB free space in the current directory. For the same reason, you will need at least 8 x 1024 MB = 8 GB memory to run the export, because DDBExport holds the intermediate data in memory before flushing to disk. 

In general, DDBExport can achieve / consume more RCU with bigger items. This is because a Scan API call returns a fixed size result set (1 MB), when the items are large, we need less number of iterations for a batch of items. When the item size is approaching 400 KB (which is the maximum size of an item in DynamoDB), a single process can achieve over 3200 RCU, which is approximately 25 MB/s. With 4 processes, you can achieve approximately 13000 RCU or 100 MB/s. When the items are small, the performance is expected to be worse. 

## Data Format

DDBImport/DDBExport uses regular JSON data format, one item per line, as shown in the example below. This allows the data to be used directly in other use cases, for example, AWS Glue and AWS Athena. 

~~~~
{"hash": "ABC", "range": "123", "val_1": "ABCD"}
{"hash": "BCD", "range": "234", "val_2": 1234}
{"hash": "CDE", "range": "345", "val_1": "ABCD", "val_2": 1234}
~~~~

The JSON data must include the primary key of your DynamoDB table. In the above-mentioned example, attribute "hash" is the hash key and attribute "range" is the range key.

We also provide a data generation utility GenerateTestData.py for testing purposes. 

Usage:

~~~~
python GenerateTestData.py -c <item_count> -f <output_file>
~~~~
  
Example:

~~~~
python GenerateTestData.py -c 1000000 -f test.json
python DDBImport.py -r us-east-1 -t TestTable -p 8 -c 20000 -s test.json 
~~~~


## Performance and Cost Considerations for DDBImport


If the data to be imported is large, it is recommended that the data be split into multiple JSON files (in a folder) instead of a single JSON file. This avoids fitting all the data into memory at once. This can be done with the **split** command in Linux. Below is an example on how to achieve this.

~~~~
$ ls -l *.json
-rw-rw-r-- 1 ec2-user ec2-user 167482559 Aug 12 09:19 test.json

$ more test.json | wc -l
1000000

$ split -dl 100000 --additional-suffix=.json test.json test                                                                  

$ ls -l *.json
-rw-rw-r-- 1 ec2-user ec2-user  16748382 Aug 12 09:21 test00.json
-rw-rw-r-- 1 ec2-user ec2-user  16748294 Aug 12 09:21 test01.json
-rw-rw-r-- 1 ec2-user ec2-user  16748330 Aug 12 09:21 test02.json
-rw-rw-r-- 1 ec2-user ec2-user  16748069 Aug 12 09:21 test03.json
-rw-rw-r-- 1 ec2-user ec2-user  16748405 Aug 12 09:21 test04.json
-rw-rw-r-- 1 ec2-user ec2-user  16748386 Aug 12 09:21 test05.json
-rw-rw-r-- 1 ec2-user ec2-user  16748401 Aug 12 09:21 test06.json
-rw-rw-r-- 1 ec2-user ec2-user  16748051 Aug 12 09:21 test07.json
-rw-rw-r-- 1 ec2-user ec2-user  16748289 Aug 12 09:21 test08.json
-rw-rw-r-- 1 ec2-user ec2-user  16747952 Aug 12 09:21 test09.json
-rw-rw-r-- 1 ec2-user ec2-user 167482559 Aug 12 09:19 test.json

$ rm test.json
~~~~


## Performance and Cost Considerations for DDBExport

When exporting a DynamoDB table at TB scale, you might want to run DDBExport on an EC2 instance with both good network performance and good disk I/O capacity. The I3 instance family becomes a great choice for such use case. The following test results are done with a DynamoDB table with 6.5 TB data. There are over 37 million items in the table, with each item being around 200 KB. The EC2 instances are i3.8xlarge, i3.16xlarge and i3en.24xlarge with Amazon Linux 2. A RAID0 device is created with all the instance-store volumes to provide the best disk I/O capacity. 

On i3.8xlarge, create a RAID0 device with 4 instance-store volumes:

~~~~
sudo mdadm --create /dev/md0 --level=0 --name=RAID0 --raid-devices=4 /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1
~~~~

On i3.16xlarge, create a RAID0 device with 8 instance-store volumes:

~~~~
sudo mdadm --create /dev/md0 --level=0 --name=RAID0 --raid-devices=8 /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1 /dev/nvme4n1 /dev/nvme5n1 /dev/nvme6n1 /dev/nvme7n1
~~~~

On i3en.24xlarge, create a RAID0 device with 8 instance-store volumes:

~~~~
sudo mdadm --create /dev/md0 --level=0 --name=RAID0 --raid-devices=8 /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1 /dev/nvme4n1 /dev/nvme5n1 /dev/nvme6n1 /dev/nvme7n1 /dev/nvme8n1 
~~~~

After that, create EXT4 file system without lazy initialization, mount the RAID0 device for writing. On i3.8xlarge, we use 32 processes. On i3.16xlarge, we use 64 processes. On i3en.24xlarge, we use both 64 and 96 processes.

~~~~
# Create EXT4 file system without lazy initialization
sudo mkfs.ext4 -E lazy_itable_init=0,lazy_journal_init=0 /dev/md0
# Mount the RAID0 device and change the ownership to ec2-user
sudo mkdir /data
sudo mount /dev/md0 /data
sudo chown -R ec2-user:ec2-user /data
# Time the DDBExport process
cd /data
time python ~/DDBImportExport/DDBExport.py -r <region> -t <table> -p <processes> -c <capacity> -s 1024 -d <destination>
~~~~

The following table summaries the tests peformed with DDBExport. Here RCU refers to the provisioned RCU on the table, which is also used in the **-c** option for DDBExport.

| ID | RCU | Instance | vCPU | Memory | SSD Disks | Network | Processes | Output |
|---|---|---|---|---|---|---|---|---|
| 1 | 112000 | i3.8xlarge | 32 | 244 GB | 4 x 1900 GB | 10 Gbps | 32 | HD |
| 2 | 112000 | i3.8xlarge | 32 | 244 GB | 4 x 1900 GB | 10 Gbps | 32 | S3 |
| 3 | 192000 | i3.16xlarge | 64 | 488 GB | 8 x 1900 GB | 25 Gbps | 64 | HD |
| 4 | 192000 | i3.16xlarge | 64 | 488 GB | 8 x 1900 GB | 25 Gbps | 64 | S3 |
| 5 | 192000 | i3en.24xlarge | 96 | 768 GB | 8 x 7500 | 100 Gbps | 64 | HD |
| 6 | 192000 | i3en.24xlarge | 96 | 768 GB | 8 x 7500 | 100 Gbps | 64 | S3 |
| 7 | 192000 | i3en.24xlarge | 96 | 768 GB | 8 x 7500 | 100 Gbps | 96 | HD |
| 8 | 192000 | i3en.24xlarge | 96 | 768 GB | 8 x 7500 | 100 Gbps | 96 | S3 |

As a comparison, we use Data Pipeline with the "Export DynamoDB table to S3" template to perform the same export. Data Pipeline launches an EMR cluster to do the work, and automatically adjust the number of core nodes to match the provisioned RCU on the table. By default, the m3.xlarge instance type is used, with up to 8 containers on each core node. The following table shows the time needed to perform the export with Data Pipeline.

| ID | RCU | Instance | vCPU | Memory | Nodes | Containers | Output |
|---|---|---|---|---|---|---|---|
| 9 | 112000 | m3.xlarge | 4 | 15 GB | 1 + 94 | 749 | S3 |
| 10 | 192000 | m3.xlarge | 4 | 15 GB | 1 + 160 |  1277 | S3 |

The following table summarizes the execution time and execution cost for the above-described tests, using on-demand pricing in the us-east-1 region. 

| ID | EC2/EMR Price | RCU Price | Consumed RCU | Time | Cost |
|---|---|---|---|---|---|
| 1 | $2.496 / hour | $14.56 / hour | 72000 | 205 minutes | $58.27 |
| 2 | $2.496 / hour | $14.56 / hour | 62000 | 239 minutes | $67.94 |
| 3 | $4.992 / hour | $24.96 / hour | 140000 | 107 minutes | $53.41 |
| 4 | $4.992 / hour | $24.96 / hour | 100000 | 159 minutes | $79.37 |
| 5 | $10.848 / hour | $24.96 / hour | 170000 | 96 minutes | $57.29 |
| 6 | $10.848 / hour | $24.96 / hour | 140000 | 110 minutes | $65.64 |
| 7 | $10.848 / hour | $24.96 / hour | 192000 | 83 minutes | $49.53 |
| 8 | $10.848 / hour | $24.96 / hour | 192000 | 80 minutes | $47.74 |
| 9 | $31.588 / hour | $14.56 / hour | 112000 | 136 minutes | $104.60 |
| 10 | $53.530 / hour | $24.96 / hour | 192000 | 84 minutes | $109.89 |

- Test 1 fails with insufficient storage capacity. The size of the DynamoDB table is 6.8 TB. The RAID0 device only has 6.6 TB available space.
- Test 2 is successful. With S3 as the output destination, only 32 x 1024 MB = 32 GB storage space is required for the intermediate data.
- Test 3 is successful. The RAID0 device offers 14 TB available space, which is sufficient to hold the data.
- Test 4 takes more time than test 3. With S3 as the output destination, each sub-process alternates between DynamoDB Scan and S3 PutObject operations, both producing a significant pressure on the network. This alternative workload pattern slows down the export process.
- The same behavior is observed in tests 5 and 6. Since i3en.24xlarge has much faster network (100 Gbps throughput), tests 5 and 6 achieve better performance as compared to tests 3 and 4, with the same number of sub-processes.
- Tests 7 and 8 complete at the approximately same time. With sufficient processing capacity, network bandwidth, disk I/O capacity, and concurrency, the provisioned RCU for the export now becomes the bottleneck. In this case, using S3 as the output destination achieves the same performance as using HD as output destination.
- In tests 7 and 8, 96 sub-processes are capable of driving more than 192000 RCU. The QoS module is effective in preventing DDBExport from consuming more RCU than allowed.

Comparing test 8 with test 10, DDBExport achieves some slight speed-up (5%) with significant cost reduction (56%), as compared to Data Pipeline. Also, Data Pipeline is a much more complicate solution, with multiple AWS services involved, and a large number of nodes in a cluster. DDBExport achieves this with a single command line on a single EC2 instance. As such, DDBExport is capable of exporting DynamoDB tables at TB scale, meeting both cost and deadline constraints. 

DDBExport has been tested with a DynamoDB table with 48 TB data (with 266 million items). We run DDBExport on i3en.24xlarge with 96 sub-processes. The provisioned RCU on the DynamoDB is 192000. The export is completed in 532 minutes. 


## Additional Notes

For a long running DDBExport with a high level of concurrency, it is normal to see the following error message. This occurs when the temporary credential obtained from the IAM role expires. DDBExport will automatically obtain new temporary credential from the IAM role. With a high level of concurrency, the credential renewal might fail in some sub-processes. DDBExport has a retry mechanism to make the renewal eventually successful.

~~~~
Worker_0063: Error when retrieving credentials from iam-role: Credential refresh failed, response did not contain: access_key, secret_key, token, expiry_time
Worker_0063: DynamoDB Scan attempt 1 failed.
~~~~