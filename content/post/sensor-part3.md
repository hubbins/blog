+++
date = "2019-06-06T12:00:00-05:00"
title = "Building an IoT temperature sensor (Part 3 of 3)"
draft = false
tags = [ "Development", "IoT", "RaspberryPi", "Python", "AWS" ]
categories = [ "Development" ]
comments = true
+++
Build the AWS services to monitor the temperature and detect failures.
<!--more-->

We want to accomplish a few main goals when developing the AWS services:

* Detect if the temperature falls beneath a certain threshold.  If so, notify a number of people via SMS.
* Detect if the temperature sensor stops sending readings due to some malfunction.  Hardware, power, no wifi, etc...  If so, notify the admin via SMS.
* Create a serverless environment with minimal running monthly costs.

# Detecting sensor failure

What I intend to do is each time a sensor sends an update, I will update a file in an S3 bucket with the timestamp of the temperature reading.  The key will be the sensor id.  

Then I will have a process to periodically check the timestamp in the S3 bucket and if it's older than some number of minutes, that means we haven't received a timely update and notify the admin.  

We can accomplish this using a Cloudwatch event and a lambda function.  The lambda function will be fired by the Cloudwatch event every 15 minutes, passing the sensor id as a parameter.  The lambda will check the timestamp for the respective sensor by finding the value in the S3 bucket.  If the timestamp is older than, say, 30 minutes, then notify the admin.

# Detecting temperature below threshold value

We will have a lambda, triggered by API Gateway, that reads a temperature from a sensor.  First, it writes the current timestamp to the S3 bucket for this sensor, to show the sensor is active.  Next, see if the temperature has fallen below a certain temperature.  If so, then notify a number of people (in this case, the condo board members) via SMS.  The SMS message will indicate the sensor id and the temperature reading.  

# Setting up AWS

Let's setup a few simple things first:

* Create an S3 bucket, using default access (non-public), called "home-monitor".
* Create a role "home-monitor-role" and attach the policies: AWSLambdaExecute (which includes S3) and AmazonSNSFullAccess

These policies are pretty wide open.  Sufficient for my purpose, but locking them down to only specific buckets and SNS actions is a good idea.  

# Sensor failure lambda

Let's take a look at the sensor failure lambda code.  This is periodically called by a Cloudwatch event.
```python
import json
import os
import datetime
import boto3
from dateutil import tz

def lambda_handler(event, context):
    
    # get sensor id from event
    sensor_id = str(event)
    # get time zones to convert message to local time
    from_zone = tz.gettz('UTC')
    to_zone = tz.gettz('America/Chicago')

    # read and parse the latest timestamp for this sensor
    s3_client = boto3.client("s3")
    obj = s3_client.get_object(Bucket=os.environ["BUCKET"], Key=sensor_id)
    timestamp_str = obj["Body"].read().decode('utf-8')
    timestamp_sensor = datetime.datetime.strptime(timestamp_str, "%Y-%m-%dT%H:%M:%S.%f")
    
    timestamp_now = datetime.datetime.utcnow()
    timestamp_30 = timestamp_now - datetime.timedelta(seconds=1800)
    
    # if timestamp in bucket is more than 30 minutes old, some failure, notify
    if (timestamp_30 > timestamp_sensor):
        # send a text message to each phone number in the list
        phone_numbers = os.environ["PHONE"].split("|")
        
        # convert sensor time to local time
        local_time = timestamp_sensor.replace(tzinfo=from_zone)

        sns_client = boto3.client("sns")

        for phone_number in phone_numbers:
            response = sns_client.publish(
                PhoneNumber=phone_number, 
                Message=sensor_id + " failed, last check in " + str(local_time.astimezone(to_zone)),
            )
    
    return {
        'statusCode': 200,
        'body': json.dumps(event)
    }
```

Here are the steps:

* Get the sensor id from the event passed in by the Cloudwatch event
* Get the sensor's latest update timestamp from the S3 bucket
* If the sensor's latest update is more than 30 minutes old, get a list of phone numbers from a lambda environment variable and send a text message indicating the sensor id that failed and its last checkin time

Then setup a Cloudwatch event that triggers this lambda every 15 minutes, passing the sensor id as a parameter (Constant (JSON text)) with a value such as "garage" (with quotes).  You can also add multiple targets to the same Cloudwatch event, say with different sensor id's.  This way, you can reuse a single Cloudwatch event to track multiple sensors.  

# Check temperature

Let's look at the lambda code that handles the temperature threshold check:
```python
import json
import os
import datetime
import boto3

def lambda_handler(event, context):
    
    result = {}
    
    # get temperature threshold
    threshold = float(os.environ["THRESHOLD"])
    
    # get temperature and sensor_id from POST body
    payload = json.loads(event["body"])
    temperature = float(payload["temperature"])
    sensor_id = payload["sensor_id"]
    
    # write current timestamp to S3 bucket for this sensor
    timestamp = datetime.datetime.utcnow().isoformat()
    
    s3_client = boto3.client("s3")
    s3_client.put_object(Body=timestamp, Bucket=os.environ["BUCKET"], Key=sensor_id)
    
    # if temperature is less than threshold, send a message with sensor name and temperature
    if temperature < threshold:
        sns_client = boto3.client("sns")
        
        # send a text message to each phone number in the list
        phone_numbers = os.environ["PHONE"].split("|")

        for phone_number in phone_numbers:
            response = sns_client.publish(
                PhoneNumber=phone_number, 
                Message=sensor_id + " " + str(temperature) + " degrees",
            )
        
        result["sns"] = response    # return last text message result
    

    result["timestamp"] = timestamp
    result["temperature"] = temperature
    
    return {
        'statusCode': 200,
        'body': json.dumps(result)
    }
```

Here are the steps:

* Get the temperature threshold from a lambda environment variable, such as 40
* Get the sensor payload and extract the temperature and the sensor id
* Write the current timestamp to the S3 bucket for this sensor id
* If the temperature falls beneath the threshold, get a list of phone numbers from a lambda environment variable, and send a text message

Create an API Gateway trigger for this lambda, with an API key.  Note the API key for use in the code that runs on the Pi.  

# Summary

We now have all the pieces in place.  

We have the Pi running, sending a temperature reading to API Gateway every 15 minutes.  

We have a lambda triggered by API Gateway that updates the latest active timestamp for the sensor.  Then if the reported temperature falls beneath the threshold, notifies a number of people via SMS.  

We have a lambda triggered by a Cloudwatch event that detects sensor failure and notifies the administrator.  

I don't know what the monthly AWS bill will be to run this.  We're talking about less than 100 API Gateway requests per day, half that many additional lambda request for failure checks, and a tiny amount of S3 storage.  Should be very minimal.  We'll see!

Overall, a fun project that introduced basic sensor usage on the Pi, some basic Pi programming, and some basic IoT with AWS web services using Python.  Also allowed me to brush up on my terrible soldering skills!  


