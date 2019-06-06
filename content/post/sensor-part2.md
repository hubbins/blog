+++
date = "2019-06-05T19:00:00-05:00"
title = "Building an IoT temperature sensor (Part 2 of 3)"
draft = false
tags = [ "Development", "IoT", "RaspberryPi", "Python", "AWS" ]
categories = [ "Development" ]
comments = true
+++
Building the Pi code to post the temperature reading to the cloud.
<!--more-->

Once the circuit is built, then writing the code to get the temperature value is very straightforward.  

The big benefit of using this particular digital temperature sensor is that the Pi has support for it directly.  Because the Pi handles the low-level communication with the sensor, we simply just have to read and parse a text file.  We don't have to bother with any low-level GPIO programming.

So we'll need a python app to do two things:  
* Read the text file and parse the temperature value
* Send the temperature value to a remote http endpoint

Let's take a look at the code:
```python
import os
import glob
import time
import urllib.request
import json

# define the AWS endpoint and API key
endpoint = "https://rpx74u5erj.execute-api.us-east-1.amazonaws.com/default/temperatureSensor"
apikey = "8MKmctQXTC7tm5C2svuq91a9lTkqmis1xxxxxx"

# unique sensor id
sensor_id = "garage"

# folder containing the text file
base_dir = '/sys/bus/w1/devices/'
device_folder = glob.glob(base_dir + '28*')[0]
device_file = device_folder + '/w1_slave'

# read the lines of the text file into an array
def read_temp_raw():
    f = open(device_file, 'r')
    lines = f.readlines()
    f.close()
    return lines

# parse the temperature from the next file, retrying if the sensor is not ready(?)
def read_temp():
    lines = read_temp_raw()
    while lines[0].strip()[-3:] != 'YES':
        time.sleep(0.2)
        lines = read_temp_raw()
    equals_pos = lines[1].find('t=')
    if equals_pos != -1:
        temp_string = lines[1][equals_pos+2:]
        temp_c = float(temp_string) / 1000.0
        temp_f = temp_c * 9.0 / 5.0 + 32.0
        return temp_c, temp_f


# round and print the temperature in F
temp_c, temp_f = read_temp()
temp_f = round(temp_f, 2)
print(temp_f)

# create the JSON payload to send to the http endpoint
payload = {}
payload["sensor_id"] = sensor_id
payload["temperature"] = temp_f

# POST to the endpoint with the JSON payload
req = urllib.request.Request(endpoint)
req.add_header('Content-Type', 'application/json; charset=utf-8')
jsondata = json.dumps(payload)
jsondataasbytes = jsondata.encode('utf-8')   # needs to be bytes
req.add_header('Content-Length', len(jsondataasbytes))
req.add_header("x-api-key", apikey)
response = urllib.request.urlopen(req, jsondataasbytes)
```

Note that some values such as http endpoint, api key, and sensor id are hard-coded in the script.  These values could also be kept in a configuration file, environment variables, etc...  Because this is a small and simple script, it's good enough to keep it self-contained.  This isn't production code.  

Next we copy this file (app3.py) to the home folder of the default "pi" user /home/pi.  You can use SFTP or another means to transfer the file.  VS Code has a nice plugin (SFTP) that when you save the file to your PC, it will also save the file in a remote folder on the Pi using SFTP.  

Then we can can create a cron task to run the app every 15 minutes.  Run the command "sudo crontab -e" then add this to the file:
```
*/15 * * * * python3 /home/pi/app3.py
```

In the last post in the series, we'll build the AWS services.
