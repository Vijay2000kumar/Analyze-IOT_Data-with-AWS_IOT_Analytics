# Analyze IoT Data with AWS IoT Analytics

AWS IoT Analytics automates the steps that are required to analyze data from Internet of Things (IoT) devices. It filters, transforms, and enriches IoT data before storing it in a time-series data store for analysis. You can set up the service to collect only the data you need from your devices and apply mathematical transforms to process the data. You can also enrich the data with device-specific metadata—such as device type and location—before you store it. Then, you can analyze your data by running queries using the built-in structured query language (SQL) query engine. You can also perform more complex analytics and machine learning inference. AWS IoT Analytics enables data visualization through integration with Amazon QuickSight.

## Components
You work with AWS IoT Analytics by using the following components:

- **Channels** – Channels collect and archive the raw messages from your IoT devices.
- **Pipelines** – Pipelines consume messages from channels. You can process the messages in the pipeline before storing them in a data store.
- **Data store** – A data store is a scalable repository for the messages coming from your IoT devices. You can query and filter the messages in the repository. A single data store can receive input from multiple channels and pipelines.
- **Dataset** – You retrieve data from a data store by creating a dataset. You can either create an SQL dataset or a container dataset.

## Objectives
After completing this lab, you will be able to:

- Access the AWS IoT Analytics Service in the AWS Management Console
- Create an AWS IoT Analytics channel
- Create an AWS IoT Analytics data store
- Create an AWS IoT Analytics pipeline
- Create an AWS IoT Core rule
- Query an AWS IoT Analytics data store

## Prerequisites
This lab requires:

- Access to a notebook computer with Wi-Fi and Microsoft Windows, macOS, or Linux (Ubuntu, SuSE, or Red Hat)
- For Microsoft Windows users: Administrator access to the computer
- An internet browser such as Chrome, Firefox, or IE9 (previous versions of Internet Explorer are not supported)

## Duration
This lab requires 60 minutes to complete. The lab will remain active for 120 minutes to allow extra time to complete the challenge question.

## Scenario Summary
In this lab, you are working for an urban planning agency of a major city. The city installed sensors to detect light and humidity levels, as well as temperatures. They intend to use the data to learn more about how vegetation can affect the temperature in the city. The data was collected from sensor readings every 5 minutes from five different locations. In the lab, you import data from these sensors. In an actual IoT implementation, the data sensors would send data directly to your AWS IoT Analytics and AWS IoT Core implementations. To read more about AWS IoT Analytics, see [What is AWS IoT Analytics](https://docs.aws.amazon.com/iotanalytics/latest/userguide/what-is-aws-iot-analytics.html).

## Tasks

### Task 1: Create a Channel
A channel collects and archives raw, unprocessed message data before it publishes the data to a pipeline.

1. On the AWS Management Console, on the Services menu, choose Services.
2. From the list of services, choose IoT Analytics.
3. In the navigation pane, choose Channels.
4. Choose Create a channel.
5. In the Channel ID box, enter `MyChannel`.
6. Select **Service-managed store**.
7. Choose Next.
8. On the Set ID, source, and data retention period page, choose Create Channel.

### Task 2: Create a Data Store
Pipelines store their processed messages in a data store. A data store is not a database. It is a scalable repository of your messages you can query.

1. In the navigation pane, choose Data stores.
2. On the Data stores page, choose Create a data store.
3. In the ID field, enter `my_datastore`.
4. Select **Service-managed store**.
5. For the data retention period, accept the default value of Indefinitely.
6. Choose Create data store.

### Task 3: Create a Pipeline
A pipeline consumes messages from one or more channels. It enables you to process the messages before you store them in a data store.

1. In the navigation pane, choose Pipelines.
2. Choose Create a pipeline.
3. In the Pipeline ID field, enter `my_pipeline`.
4. To select the Pipeline source, choose Edit.
5. Choose `my_channel`.
6. Choose Next.
7. Add attributes manually if necessary.
8. Choose Next until you reach the Save your processed messages in a data store page.
9. Choose Edit and select the `my_datastore` data store.
10. Choose Create pipeline.

### Task 4: Create an AWS IoT Core Rule
AWS IoT Analytics integrates with the AWS IoT Core service through a message broker. IoT devices send their data to a topic. The messages are sent using the Message Queuing Telemetry Transport (MQTT) protocol.

1. On the AWS Management Console, on the Services menu, choose Services.
2. From the list of services, choose IoT Core.
3. Choose Get started.
4. From the navigation menu, choose Act.
5. On the Rules menu, choose Create a rule.
6. In the Name box, enter `Send_IOT`.
7. Copy and paste the following SQL statement into the Rule query statement box:
    ```sql
    SELECT * FROM 'iot/aus_weather'
    ```
8. In the Set one or more actions section, choose Add action.
9. Select **Send a message to IoT Analytics**.
10. Choose Configure action.
11. Select **Manually select IoT Analytics Channel and role**.
12. Choose the channel you created earlier (`mychannel`) and the role `IoTLabAccessRole`.
13. Choose Add action and Create rule.

### Task 4.2: Configure Your Environment to Run the Python Script
You will run a Python script to load data into the `iot/aus_weather` MQTT topic. Use SSH to connect to an Amazon EC2 instance.

#### Windows Users – Use SSH to Connect
1. Download the labsuser.ppk file from the Details panel.
2. Open PuTTY and configure the session with the following settings:
    - Host Name: Use the IPv4 Public IP address of the Command Host instance.
    - Connection: Set Seconds between keepalives to 30.
    - Auth: Browse and select the labsuser.ppk file.
3. Choose Open and log in as `ec2-user`.

#### macOS and Linux Users
1. Download the labsuser.pem file from the Details panel.
2. Open a terminal window and navigate to the directory where the labsuser.pem file was downloaded.
3. Change the permissions on the key to be read-only:
    ```sh
    chmod 400 labsuser.pem
    ```
4. SSH into the EC2 instance:
    ```sh
    ssh -i labsuser.pem ec2-user@<public-ip>
    ```

### Task 4.3: Configure the Amazon EC2 Environment
1. Set the AWS CLI credentials:
    ```sh
    vim ~/.aws/credentials
    ```
    Paste the AWS CLI credentials provided in the lab details and save the file.
2. Install the AWS SDK for Python (boto):
    ```sh
    pip install boto3 --user
    ```
3. Download the dataset:
    ```sh
    curl -XPORT 'https://data.melbourne.vic.gov.au/resource/277b-wacc.json' > input_aus.json
    ```

### Task 4.4: Create the Python Script
1. Create a Python script to simulate ingesting data from IoT devices:
    ```python
    import json
    import boto3
    import fileinput
    import multiprocessing as mp
    import os

    processes = 4

    # An array of boto3 IoT clients
    IotBoto3Client = [boto3.client('iot-data') for i in range(processes)]

    def publish_wrapper(lineID, line):
        # Select the appropriate boto3 client based on lineID
        client = IotBoto3Client[lineID % 4]

        line_read = line.strip()
        print("Publish: ", os.getpid(), lineID, line_read[:70], "...")
        payload = json.loads(line_read)

        # Publish JSON data to AWS IoT
        client.publish(
            topic='iot/aus_weather',
            qos=1,
            payload=json.dumps(payload)
        )

    if __name__ == '__main__':
        pool = mp.Pool(processes)
        jobs = []
        print("Begin Data Ingestion")
        for ID, line in enumerate(fileinput.input()):
            # Create job for each JSON object
            res = jobs.append(pool.apply_async(publish_wrapper, (ID, line)))

        for job in jobs:
            job.get()

        print("Data Ingested Successfully")
    ```
2. Save the script as `upload_raw_data_iot.py`:
    ```sh
    vim upload_raw_data_iot.py
    ```

### Task 5: Create a Dataset
You retrieve data from a data store by creating a dataset.

1. On the AWS Management Console, on the Services menu, choose Services.
2. From the list of services, choose IoT Analytics.
3. In the left navigation pane, choose Data sets.
4. On the Data sets page, choose Create a data set.
5. Choose Create SQL.
6. In the ID field, enter `my_dataset`.
7. Select the `my_datastore` data store.
