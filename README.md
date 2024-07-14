

# Sales Data Projection

It is near real time Change Data Capture (CDC) sales data ingestion pipeline where eventbridge pipe capture the real time data in DynamoDB and push into the kinesis stream which delivers the data to the kinesis firehose which dumps the transformed data with the help of lambda in no batch files into amazon S3 bucket on top of it Athena table is getting created to sync data in near real time for further data analysis

## Tech stack used:
    1. Python Mock generator script
    2. DynamoDB
    3. DynamoDB stream(for CDC)
    4. Kinesis stream
    5. Eventbridge pipe (for stream ingestion)
    6. Kinesis Firehose (for batch streaming data)
    7. Lambda for transformation
    8. Athena for data analysis
    9. Amazon S3 bucket as a destination

## Architecture of data pipeline:

## Steps to design a pipeline:

### Step1 : Set up DynamoDB and DynamoDB stream
    1. Login to AWS account, search for DynamoDB and  create table as "Ordertable" with orderid is partition key
    2. Run the python mock data generation script (mock_data_generator_for_dynamodb.py) which will generate and insert the new records into the DynamoDB table
    DynamoDB table creation
  ![create_dynamodb_table](https://github.com/user-attachments/assets/c33ee3e7-5b52-4789-8917-39cd56792b61)
  
    After running mock data generation python script data get inserted into dynamoDB table
   ![data_get_inserted_into_dynamodb_table](https://github.com/user-attachments/assets/0d9c11aa-2071-4c8e-8304-c71bd3e3d3f1)


### Step2: Enable DynamoDB stream for Change data capture (CDC)
Click on Ordertable ,will find export and stream option ->go to DynamoDB streamdetails->turn on the status option

![enable_dynamodb_streaming](https://github.com/user-attachments/assets/a276f4e4-fe7a-4035-a96e-1d0189ca5eb5)

### Step3: Create Kinesis stream
Search kinesis service in AWS -> go to Data streams -> create new kinesis stream as "Kinesis-sales-order"

![create_kinesis_stream](https://github.com/user-attachments/assets/ac1f2fe1-48ae-428e-b1b8-e20092718d6f)

### Step4: Set up event bridge pipe
Need to set up event bridge pipe so that CDC can capture from DynamoDB to kinesis to do that search amazon eventbridge-> choose event bridge pipe -> create pipe with name of "dynamo-cdc-to-kinesis"-> select source as "DynamoDB" -> select latest created dynamoDb stream -> select target as "kinesis-stream"-> select stream as "kinesis-sales-order"-> select eventID as partition key,so based on partition key it will decide in which shardes messages can go -> click on create pipe -> once pipe get created by default it will create role -> click on role and attach policies to give full access for dynamoDB and kinesis -> run the mock data generation script ->will see data get added into kinesis stream -> if input data get change that also get capture due to activation of dynamoDB stream

Eventbridge pipe between DynamoDB and kinesis

![create_eventbridge_pipe_between_dynamodb_kinesis](https://github.com/user-attachments/assets/c602d52e-577d-428d-b46d-873143dc5402)

CDS get capture in kinesis(data get inserted or updated as per the source level manipulations)

![changes_get_inserted_and_modified_CDC_in_kinesis](https://github.com/user-attachments/assets/4d367a54-eef3-440c-9e52-f8a9a54024f9)

After inserting data into kinesis records will be look in below format

![message_data](https://github.com/user-attachments/assets/ffb41f40-4e15-4588-b333-9b1e0d5d8a99)


### Step5: Set up Kinesis data firehose
Search amazon data firehose -> create delivery stream -> select source as Amazon kinesis data stream -> select destination as Amazon S3 ->select already created kinesis data stream "kinesis-sales-order" -> rename firehose data stream as "kinesis-to-S3-delivery" -> will bring lambda in between for transforming data before dump into S3



### Step6: Set up Lambda
Search lambda -> create function as "stream transformation" -> copy the code present in transformation_layer_with_lambda.py ,will get data from firehose as all in single line {r1},{r2},{r3} later crawler will not able to read it properly to avoid that will add newline in between records using lambda function it will convert records like {r1}\
                                             {r2}\
                                             {r3}

to apply lambda transformation -> enable data transformation property -> select already created "stream transformation" lambda function ->create bucket of sales-data-projection-outcome -> choose created bucket as destination -> set up hints,compression and encryption setting -> we want firehose should create batch,till how long it should wait to create next new batch so set up buffer size and buffer interval according to incoming data size ->click on create firehose delivery stream
Now start the mock data script -> so it will generate data in realtime-> event bridge will capture it-> put incoming data into kinesis stream-> firehose will batch it-> lambda will do transformation and dump the transformed data into S3 bucket,refresh S3 bucket will see different output files get added in S3 

### Step7: Create athena table on top of S3 files
go to AWS Glue -> create database as "sales-data-projection-db"->create crawler to create meta table for S3 file, to do that select source as S3 bucket ->have to create custom classifier to read JSON data ,so that crawler can identify and parse json data to do that add new classifier as "json-classifier"->after all properties set, click on create crawler -> run the crawler, it will create meta data table -> now go to the athena query on table that we have already created on top of S3 bucket

Create database in glue as "sales-data-projection-db"

![create_database_glue](https://github.com/user-attachments/assets/c987668b-190a-470b-acba-e4c11226008f)

Sales-data-crawler creation

![sales-data-crawler-creation](https://github.com/user-attachments/assets/9acdb2dd-fe55-4e16-8336-23cf42b05454)

After running the above crawler meta data table gets created in Glue catalog

![meta_data_creation_after_crawler_run](https://github.com/user-attachments/assets/88fc99a8-5b81-41a1-8c30-5d806aae7ede)


Query in Athena

![athena_query_output](https://github.com/user-attachments/assets/1ee03640-07c8-4341-939a-208f9ea9cc10)

Here we have done realtime data sync in using near real time data analysis pipeline

### Output: Transformed data get ingested into amazon S3

![transformed-date-written-into-S3-bucket](https://github.com/user-attachments/assets/f63f08a2-7076-42aa-b063-f0b12dac607a)




