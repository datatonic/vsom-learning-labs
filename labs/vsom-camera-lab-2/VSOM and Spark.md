# VSOM and Spark

**Warning! This lab assumes you have done the Video Surveillance Operations Manager (VSOM) lab and that you have access to the server infrastructure in a private lab or if you are at an event such as Cisco Live!**

## Overview
Before proceeding with this lab, make sure that you have completed the following steps in the previous Video Surveillance Operations Manager (VSOM) lab

1. [x] Retrieve a token for gaining authorized access to the VSOM Server
2. [x] Retrieve a list of cameras that are currently connected to the VSOM Server
3. [x] Retrieve a location tree
4. [x] Create a device template
5. [x] Query a job to determine it's status
6. [x] Apply a device template to a camera
7. [x] Set the motion configuration of a camera
8. [x] Tell a camera to begin recording
9. [x] Retrieve a snapshot of a recording at a specific time
10. [x] Tell a camera to stop recording

This lab will also assume that you have a [Cisco Spark Account](https://web.ciscospark.com/) and that you have a Cisco Spark access token.

What We Will Be Doing
----------------------

In this lab we will be writing [RESTful API](http://stackoverflow.com/questions/671118/what-exactly-is-restful-programming) calls using [Python]() to intereact with both the [VSOM](http://www.cisco.com/c/en/us/products/physical-security/video-surveillance-operations-manager-software/index.html) and [Spark](https://developer.ciscospark.com/getting-started.html) API's.

We will modify the code that we wrote in the **Video Surveillance Operations Manager (VSOM)** lab

* instead of saving the snapshot as a local file, we will upload that file to an external server
* we will make an API call to [Cisco Spark](https://developer.ciscospark.com/getting-started.html), which will then post the uploaded snapshot to a [Cisco Spark](https://developer.ciscospark.com/getting-started.html) room


***Let's begin!***


Setup
----------------
If you have done the previous lab, your working directory should look similiar to this

 * train_example (previous VSOM Lab)
 	* example.py
 	* APIBase.py
 	* VSOM
 		* Client.py
 		* API
 			* Authentication.py
 			* Camera.py
 			* CameraBulkOps.py
 			* CameraRecording.py
 			* DeviceTemplate.py
 			* Job.py
 			* Location.py

Let's creat another folder named Spark, directly underneath train_example

 * Spark
 	* API
 	* Client.py

Spark
------------

Inside of our Spark folder, add this to our **Client.py** file

~~~python
import json
import APIBase

def new(uri, body={}, token=None):
  headers = {
    "Authorization": "Bearer " + token,
    "Content-Type": "application/json; charset=utf-8"
  }
  return APIBase.Client("https://api.ciscospark.com/v1", uri, headers, json.dumps(body))
~~~

Inside of our Spark/API folder, add a new file named **Messages.py**

~~~python
#### Messages.py will have methods for interacting with 
#### the /message endpoint
import Spark.Client

def send(token, text, files, room_id, to_person_id, to_person_email):
  body = {}
  if text != None:
    body["text"] = text
  if files != None:
    body["files"] = files
  if room_id != None:
    body["roomId"] = room_id
  if to_person_id != None:
    body["toPersonId"] = to_person_id
  if to_person_email != None:
    body["toPersonEmail"] = to_person_email
  client = Spark.Client.new("/messages", body, token)
  return client.post_it()
~~~

Now let's try and see if we can use the Spark API's to send a message to someone. Inside of our **example.py** file, add this line at the very top

~~~python
import Spark.API.Messages       as Messages
~~~

And then let's add this to our vsom_run() method

~~~python
def vsom_run():
  spark_token      = ""    # your spark user token
  reciepient_email = ""    # the spark user's email that will be recieiving the message
  token = vsom_token()
  res   = get_snapshot(token)
  files = {'upload_file': res.content}
  r     = requests.post("http://128.107.70.19/images/tmp", files=files)
  json  = r.json()
  url   = "http://128.107.70.19" + json["uri"]
  res   = Messages.send(spark_token, "Testing", [url], None, None, reciepient_email )
  pretty_print(res)
~~~

Now in the cmd line, run the following command 

for linux/mac

~~~
python3 example.py 
~~~
for windows

~~~
py -3 example.py
~~~

If succesful, you should have a snapshot of one of the camera recordings sent to you inside of a spark message
