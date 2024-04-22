# Create a Lambda function that listens for the creation of a .csv file in an S3 bucket and then uploads the content of the file into an RDS MySQL database.

Requirements:

 - The credentials of the database should not be hardcoded into the Lambda function.

Prerequisite:

Create an S3 bucket containing a zipped pymysql module.
Set up an EC2 instance that will be used to connect to the MySQL database to confirm that the file has been dumped into the database.
 - Use user data to bootstrap updating the instance.
 - Run the following commands after connecting to the instance:

```
sudo apt update -y
sudo apt install python3.11 -y
sudo apt install python3-pip
sudo su
sudo pip3 install pymysql
```

Install the zip application:
`sudo apt install zip -y`

Connect to your AWS user by installing awscli into your instance.
Create an access key for your IAM user.

Configure AWS CLI:
` aws configure ` 
After connecting to your user programmatically, upload the zipped pymysql into an S3 folder: 

`aws s3 cp pymysql.zip s3://bucket-name/path/to/upload/`

Create a VPC and create RDS MySQL inside it.
 - Create a VPC with 2 public and private subnets.
 - Create your MySQL database inside the VPC.
 - Make sure the database is publicly accessible.
 - Configure the RDS security group to allow traffic coming from your EC2 instance.

Log into your database and create a database and a table inside the database:


Create an IAM role giving permissions to S3, CloudWatch, and RDS.
Create your Lambda function.
Attach the newly created role to the Lambda function.
Upload from S3 location:
Copy pymysql URL from S3 and paste it.
Create a file in the Lambda executioner folder called lambda_function.py.
Open the file and insert the code (delete any existing code).

Lambda Function Code:
python
Copy code

```
import json
import csv
import pymysql
import boto3
import logging
from io import StringIO

# Initialize AWS clients
s3 = boto3.client('s3')
secretsmanager = boto3.client('secretsmanager')

# Configure logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)


def get_database_credentials(secret_name):
    '''
    retrieve database connection details from AWS secrets manager
    '''
    response = secretsmanager.get_secret_value(SecretId=secret_name)
    secret = response['SecretString']
    return json.loads(secret)


def lambda_handler(event, context):
    try:
        # Replace 'your-secret-name' with the actual name of your secret in secrets manager
        secret_name = 'db-cred'
        secrets = get_database_credentials(secret_name)

        # Database connection details are now obtained from secrets manager
        db_host = secrets['host']
        db_user = secrets['username']
        db_password = secrets['password']
        db_name = secrets['dbname']

        # Iterate through each S3 event record
        for record in event['Records']:
            # Get bucket name and file key from the event
            bucket = record['s3']['bucket']['name']
            key = record['s3']['object']['key']

            # Check if the file is a CSV file
            if key.endswith('.csv'):
                # Retrieve the CSV file contents from S3
                response = s3.get_object(Bucket=bucket, Key=key)
                csv_data = response['Body'].read().decode('utf-8')

                # Connect to the RDS MySQL database
                conn = pymysql.connect(
                    host=db_host,
                    user=db_user,
                    password=db_password,
                    db=db_name,
                    charset='utf8mb4',
                    cursorclass=pymysql.cursors.DictCursor
                )

                # Parse the CSV data
                csv_reader = csv.DictReader(StringIO(csv_data))
                for row in csv_reader:
                    # Replace empty strings with None to represent NULL values
                    row = {k: v if v else None for k, v in row.items()}

                    columns = ', '.join(row.keys())
                    placeholders = ', '.join(['%s'] * len(row))  # Ensure commas separate placeholders
                    sql = f"INSERT INTO csv ({columns}) VALUES ({placeholders})"

                    # Extract values from the row dictionary for insertion
                    values = tuple(row.values())

                    # Insert row into the database
                    with conn.cursor() as cursor:
                        cursor.execute(sql, values)
                    conn.commit()

                conn.close()

                logger.info('CSV data uploaded to RDS MySQL database successfully.')

        return {
            'statusCode': 200,
            'body': json.dumps('CSV data uploaded to RDS MySQL database successfully.')
        }
    except Exception as e:
        logger.error(f'Error: {str(e)}')
        return {
            'statusCode': 500,
            'body': json.dumps(str(e))
        }
```

## Additional Steps:
Add a trigger from another S3 bucket to the Lambda function. 

This will be the S3 bucket you upload your CSV file to.
Whitelist traffic entering the RDS database.
Change timeout limit to about 30 seconds.

