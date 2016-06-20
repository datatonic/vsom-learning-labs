# Video Surveillance Operations Manager (VSOM)
**Warning! This lab assumes you have access to the server infrastructure in a private lab or if you are at an event such as Cisco Live!**

## Overview
**Cisco Physical Security solutions** provide broad capabilities in video surveillance, IP cameras, electronic access control, and groundbreaking technology that converges voice, data, and physical security in one modular platform. Cisco Physical Security solutions enable customers to use the IP network as an open platform to build more collaborative and integrated physical security systems while preserving their existing investments in analog-based technology.

**Cisco ® Video Surveillance Media Server** is the core component of the Cisco Video SurveillanceZ Manager.
By using the power and advanced capabilities of IP networks, Cisco Video Surveillance Media Server software allows applications, users, cameras, and storage to be added over time. As a result, the software provides unparalleled video surveillance system flexibility and scalability.

![VSOM image] (assets/images/data_sheet_C78-490677-00-1.jpg)



Useful Terminology
----------------------
**VSOM** - Cisco ® Video Surveillance Operations Manager

**Location Tree** - A representation of the VSOM's file hierachy

**Device Template** - Templates simplify camera configuration by defining the image quality, recording schedule and other attributes used by a set of cameras.

**Job** - Many user actions (such as editing a camera template) trigger a Job that must be completed by the Cisco VSM system. These Jobs are completed in the background so you can continue working on other
tasks while the Job is completed. Although most Jobs are completed quickly, some actions (such as
modifying a camera template) may take longer to complete if they affect a large number of devices.

**Motion Configuration** - The settings that define the amount and type of motion required to trigger a motion detection event.

**Motion Detection Event** - Cameras that support motion detection can trigger actions or record video when motion occurs in the camera’s field of view. For example, a camera pointed at the rear door of a building can record a motion
event if a person walks into the video frame. A motion event can also trigger alert notifications, a
camera’s PTZ controls, or a URL action on a third party system.


What We Will Be Doing
----------------------
In this lab we will be writing [RESTful API](http://stackoverflow.com/questions/671118/what-exactly-is-restful-programming) calls using [Python3](https://www.python.org/) to intereact with the [VSOM](http://www.cisco.com/c/en/us/products/physical-security/video-surveillance-operations-manager-software/index.html) API.

We will use the VSOM API to do the following things

* Retrieve a token for gaining authorized access to the VSOM Server
* Retrieve a list of cameras that are currently connected to the VSOM Server
* Retrieve a location tree
* Create a device template
* Query a job to determine it's status
* Apply a device template to a camera
* Set the motion configuration of a camera
* Tell a camera to begin recording
* Retrieve a snapshot of a recording at a specific time
* Tell a camera to stop recording

***Let's start consuming API's!***


Python
-----------------------

Let's get started with creating our working directory. Open up a terminal and create a new directory named **train_example**. Inside of this directory you'll need to create 3 python files.


~~~
1. __init__.py (this should be an empty file)
2. APIBase.py (this will contain a python class that we will use to simplify http requests/responses
3. example.py (we will use this file to run our examples)
~~~


Inside of our **APIBase.py**, add the following code

~~~python

import requests
import json

class Client(object):
    def __init__(self, host, uri, headers={}, body={}, verify=True):
      self.__host  = host
      self.uri     = uri
      self.body    = body
      self.headers = headers
      self.verify  = verify
    
    # we will use get_it() for sending a get request
    # and returing a json representation of the corresponding http response's body
    def get_it(self):
      r = self.get()
      return r.json()

    # we will use post_it() for sending a post request
    # and returing a json representation of the corresponding http response's body
    def post_it(self):
      r = self.post()
      return r.json()

    # we will use post() for sending a post request
    # and returing the corresponding http response
    def post(self):
      return requests.post(self.__host + self.uri, headers=self.headers, data=self.body, verify=self.verify)
    
    # we will use post() for sending a post request
    # and returing the corresponding http response   
    def get(self):  
      return requests.get(self.__host + self.uri, headers=self.headers, verify=self.verify)

~~~

Now let's create a new folder named **VSOM**. The VSOM folder will contain all code that's relevant to the VSOM API. 

VSOM API
----------------------

To view the VSOM API's please visit: 


[https://vsom.cisco.local/ismserver](https://vsom.cisco.local/ismserver)

This is the location of the VSOM server (your VSOM server's IP address may be different)


###Retrieve an access token

Before you are able to interact with the VSOM API, you need to login into the VSOM system. Based on the type and group that the user belongs to, different levels of access will be granted. Upon a successful login, VSOM will respond with a token that you can then use in subsequent requests.

Inside of our VSOM folder, let's create a new file named **Client.py**. 

~~~python
import json
import APIBase

# this is the uri for interacting with the VSOM JSON APIs
# (your VSOM server's IP address may be different)
VSOM_JSON_URI = 'https://10.10.120.70/ismserver/json'

def new(uri, body={}, token=None):
  headers = {'x-ism-sid': token}
  return APIBase.Client(VSOM_JSON_URI, uri, headers, json.dumps(body), False)
~~~

Inside of our VSOM folder, let's create a new folder named API. Inside of our new API folder, let's create two new files. 

~~~
1. __init__.py (this should be an empty file)
2. Authentication.py (this will contain all code relavant to the /authentication endpoint)
~~~

Inside of our **Authentication.py** file, add the following code.

~~~python
import VSOM.Client

def login(name, password):
  body = {
    "username": name,
    "password": password
  }
  client = VSOM.Client.new("/authentication/login", body)
  return client.post_it()
~~~

Now let's see if we can successfuly retrieve an access token. Inside of our example.py file, let's add the following code.

~~~python
import json

import VSOM.API.Authentication  as Authentication
VSOM_USERNAME = ""  # a valid vsom user name
VSOM_PASSWORD = ""  # a valid vsom user password

def pretty_print(data):
  print (json.dumps(data, indent=4, separators=(',', ': ')))

def vsom_token():
  res = Authentication.login(VSOM_USERNAME, VSOM_PASSWORD)
  return res["data"]["uid"]
  
def vsom_run():
  print(vsom_token())

vsom_run()
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

If successful, you should see the token printed to your console.

###Retrieve a list of cameras that are currently connected to the VSOM Server

Now that we are able to successfuly login and retrieve an access token, we can start making API calls to the VSOM server.

Inside of our VSOM/API folder, let's create a new file named **Camera.py** and add the following code.

~~~python
#### Camera.py will have methods for interacting with the /camera endpoint
import VSOM.Client

def get_cameras(token):
  body = {
    "filter": {
      "byObjectType": "device_vs_camera",
      "pageInfo": {
        "start": "0",
        "limit": "100"
      }
    }
  }
  client =  VSOM.Client.new("/camera/getCameras", body, token)
  return client.post_it()
~~~

Now let's add this to the top of our **example.py** file

~~~python
import VSOM.API.Camera          as Camera
~~~

And let's modify our vsom_run method inside of our example.py file

~~~python
def vsom_run():
  token = vsom_token()
  res   = Camera.get_cameras(token)
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


The output should be similiar to [get-cameras.txt](assets/get-cameras.txt)


###Retrieving a location tree

Inside of the VSOM/API folder, create a new file named **Location.py**

Add this to our **Location.py** file

~~~python
#### Location.py will have methods for interacting with the /location endpoint
import VSOM.Client

def get_tree_root(token):
  body = {
    "treeFilter": {
      "objectTypes": [],
      "depth": 0,
      "getLocalTreeOnly": False
    }
  }
  client = VSOM.Client.new("/location/getLocationTree", body, token)
  return client.post_it()
~~~

Add this to the top of our **example.py** file

~~~python
import VSOM.API.Location        as Location
~~~

let's modify our ***vsom_run*** method inside of our **example.py** file

~~~python
def vsom_run():
  token = vsom_token()
  res   = Location.get_tree_root(token)
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

The output should be similiar to [get-location-root.txt](assets/get-location-root.txt)

###Create a device template

Now let's create a device template so we can assign it to one of our cameras.
Inside of the VSOM/API folder, create a new file named **DeviceTemplate.py**
And add the following to it

~~~python
#### DeviceTemplate.py will have methods for interacting with the /devicetemplate endpoint

import VSOM.Client

def create_default_template(token, name, description, vsom_uid, location_ref):
  body = {
    "template":{
      "systemDefined": False,
      "shared": True,
      "generic": True,
      "numAssocDevices": 0,  
      "ownerLocationRef": location_ref,     # a reference to the location
      "videoStreams": [{
        "streamNum": 1,
        "viewable": True,
        "motion_configured": True,
        "recordable": True,
        "videoStreamProfile": {
          "videoQuality": "medium",
          "format": "NTSC",
          "ltsRetentionDays": 0,
          "suspendableProxy": False
        }
      }],
      "recordings": [{
        "recordingType": "LOOP",
        "streamNum": 1,
        "duration": 86400,
        "expireTime": 1,
        "eventExpireTime": 2,
        "storageEstimation": True,
        "startImmediate": True,
        "recordIframe": False,
        "archiveToLTS": False,
        "ltsRetentionTime": 0,
        "zeroVideoLossEnabled": False
      }],
      "participateInFailover": False,
      "preBuffer": 0,
      "postBuffer": 0,
      "lastModified": 0,
      "enableRecordNow": True,
      "mergeRecordings": False,
      "name": name,               			# name of the device template
      "tags": "tags",
      "description": description,			# description of the device template
      "vsomUid": vsom_uid,					# uid of the device template
      "objectType": "vs_deviceTemplate"
    }
  }
  client = VSOM.Client.new("/devicetemplate/createDeviceTemplate", body, token)
  return client.post_it()
~~~

Add this to the top of our **example.py** file

~~~python
import VSOM.API.DeviceTemplate  as DeviceTemplate
~~~

then add this to our **example.py** file, right above the ***vsom_run*** declaration

~~~python
def create_template(token):
  tree_root = Location.get_tree_root(token)
  vsom_uid  = tree_root["data"]["vsomUid"]
  name      = ""     # name of your template
  desc      = ""     # description of your template
  location  = {
    "refUid":         tree_root["data"]["childGroups"][0]["uid"],
    "refName":        tree_root["data"]["childGroups"][0]["name"],
    "refObjectType":  tree_root["data"]["childGroups"][0]["objectType"],
    "refVsomUid":     tree_root["data"]["childGroups"][0]["vsomUid"]
  }
  return DeviceTemplate.create_default_template(token, name, desc, vsom_uid, location)

~~~

Finally, let's modify our ***vsom_run*** method

~~~python
def vsom_run():
  token = vsom_token()
  res   = create_template(token)
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

The output should be similiar to [create-default-template.txt](assets/create-default-template.txt)

When attempting to create a device template, VSOM will run the job in the background. We can track the progress of the job by sending a post request to the /job/getJobStatus endpoint and by providing a reference to the job within the post body. 

###Query a job to determine it's status
Inside of the VSOM/API folder, create a new file named **Job.py**

Add this to our **Job.py** file

~~~python
#### Job.py will have methods for interacting with the /job endpoint
import VSOM.Client

def get_status(token, job_reference):
  body   = { "jobRef": job_reference }
  client = VSOM.client.new("/job/getJobStatus", body, token)
  return client.post_it()
~~~

Add this to the top of our **example.py** file

~~~python
import VSOM.API.Job             as Job
import time
~~~

then add this to our **example.py** file, right above the ***vsom_run*** declaration

~~~python
def get_job(token, job_reference, timer):
  result = Job.get_status(token, job_reference)
  if result["status"]["errorType"] == "SUCCESS":
    if ( int(result["data"]["numberOfRunningSubJobs"]) != 0 or 
    	 int(result["data"]["numberOfPendingSubJobs"]) != 0):
      print("still running...")
      pretty_print(result)
      time.sleep(timer)
      get_job(token, job_reference, timer*2) # recursively retrieve info on the job (with exponential back off)
    else:
      pretty_print(result)
  else:
  	pretty_print(result)
  return result
  
~~~

Finally, let's modify our ***vsom_run*** method

~~~python
def vsom_run():
  token = vsom_token()
  res   = create_template(token)
  get_job(token, res["data"], 5)
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
The output should be similiar to [get-job-status.txt](assets/get-job-status.txt)

If you look closely at the http response body, you will notice that the key "numberOfFailedSubJobs" has a value of 1, while the key "numberOfCompletedSubJobs" has a value of 0. Let's try running the **example.py** file one more time, but let's modify our ***create_template*** method so that we can create a template with a unique name.

~~~python
def create_template(token):
  tree_root = Location.get_tree_root(token)
  vsom_uid  = tree_root["data"]["vsomUid"]
  #name     = ""      # name of your template
  name      = ""      # name of your new template (must be unique)
  desc      = ""      # description of your template
  location  = {
    "refUid":         tree_root["data"]["childGroups"][0]["uid"],
    "refName":        tree_root["data"]["childGroups"][0]["name"],
    "refObjectType":  tree_root["data"]["childGroups"][0]["objectType"],
    "refVsomUid":     tree_root["data"]["childGroups"][0]["vsomUid"]
  }
  return DeviceTemplate.create_default_template(token, name, desc, vsom_uid, location)
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


And confirm that the http response body has a key "numberOfCompletedSubJobs" that has a value > 0

###Apply a device template to a camera

Now that we have successfully created a template, we can attempt to apply it to a camera.
Inside of the **VSOM/API** folder, create a new file named **CameraBulkOps.py**

~~~python
#### CameraBulkOps.py will have methods for interacting with the /camerabulkops endpoint
import VSOM.Client

def associate_cameras_to_device_template(token, device_references, template_reference):
  body = {
    "deviceRefs": device_references,
    "templateRef": template_reference
  }
  client = VSOM.Client.new("/camerabulkops/associateCamerasToDeviceTemplate", body, token)
  return client.post_it()
~~~

In order to apply a template to a camera or cameras, we need a template reference and a camera reference for each of the cameras we intend on applying that template to. Let's add an additional method to our **Camera.py** file so we can can retrieve a camera by it's name as opposed to retrieving an entire list of cameras.

Add this method to our **Camera.py** file

~~~python
def get_by_name(token, name):
  body = {
    "filter": {
      "byObjectType": "device_vs_camera",
      "byExactName": name,
      "pageInfo": {
        "start": "0",
        "limit": "100"
      }
    }
  }
  client =  VSOM.Client.new("/camera/getCameras", body, token)
  return client.post_it()
~~~


Now that we can retrieve a camera by it's name, let's write another method that will allow us to do the same with a template. 

Add this method to our **DeviceTemplate.py** file

~~~python
def get_by_name(token, name):
  body = {
    "filter": {
      "byExactName": name
    }
  }
  client = VSOM.Client.new("/devicetemplate/getDeviceTemplates", body, token)
  return client.post_it()
~~~

Now we can attempt to apply a template to our camera

Let's add the following to the top of our **example.py** file

~~~python
import VSOM.API.CameraBulkOps   as CameraBulkOps
~~~

then add this to our **example.py** file, right above the ***vsom_run*** declaration

~~~python
def get_device_reference(device):
  device_reference = {
    "refUid":        device["data"]["items"][0]["alternateId"],
    "refName":       device["data"]["items"][0]["name"],
    "refObjectType": device["data"]["items"][0]["objectType"],
    "refVsomUid":    device["data"]["items"][0]["vsomUid"]
  }
  return device_reference

def apply_template(token):
  camera_name        = ""   # name of the camera
  template_name      = ""   # name of the template
  device             = Camera.get_by_name(token, camera_name)
  template           = DeviceTemplate.get_by_name(token, template_name)
  device_reference   = get_device_reference(device)
  template_reference = {
    "refUid":        template["data"]["items"][0]["uid"],
    "refName":       template["data"]["items"][0]["name"],
    "refObjectType": template["data"]["items"][0]["objectType"],
    "refVsomUid":    template["data"]["items"][0]["vsomUid"]
  }
  res = CameraBulkOps.associate_cameras_to_device_template(token, [device_reference], template_reference)
  return get_job(token, res["data"], 5)
~~~

Finally, let's modify our ***vsom_run*** method

~~~python
def vsom_run():
  token = vsom_token()
  res   = apply_template(token)
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

The output should be similiar to [apply-template.txt](assets/apply-template.txt)

###Set the motion configuration of a camera

Now that we have applied a template to a camera, we can configure the motion detection settings of a camera.

Let's create a new method so we can configure our camera so that it will be capable of detecting motion. 

Add this method to our **CameraBulkOps.py** file, inside of the CameraBulkOps module.

~~~python
def create_full_motion_windows(token, device_references):
  body = {
    "deviceRefs": device_references
  }
  client = VSOM.Client.new("/camerabulkops/createFullMotionWindows", body, token)
  return client.post_it()
~~~

then add this to our **example.py** file, right above the ***vsom_run*** declaration

~~~python
def configure_motion_window(token):
  camera_name      = ""
  device           = Camera.get_by_name(token, camera_name)
  device_reference = get_device_reference(device)
  res              = CameraBulkOps.create_full_motion_windows(token, [device_reference])
  return get_job(token, res["data"], 5)
~~~

And let's modify our ***vsom_run*** method so that it looks like this

~~~python
def vsom_run():
  token = vsom_token()
  res   = configure_motion_window(token)
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
The output should be similiar to [configure-motion-window.txt](assets/configure-motion-window.txt)

###Tell a camera to begin recording

If you have been following the learning lab up to this point, you should have completed the following tasks

1. [x] Retrieve a token for gaining authorized access to the VSOM Server
2. [x] Retrieve a list of cameras that are currently connected to the VSOM Server
3. [x] Retrieve a location tree
4. [x] Create a device template
5. [x] Query a job to determine it's status
6. [x] Apply a device template to a camera
7. [x] Set the motion configuration of a camera
8. [_] Tell a camera to begin recording
9. [_] Retrieve a snapshot of a recording at a specific time
10. [_] Tell a camera to stop recording

Inside of the VSOM/API folder, create a new file named **CameraRecording.py**

Add this to our **CameraRecording.py** file

~~~python
#### CameraRecording.py will have methods for interacting with the /cameracamerarecording endpoint
import VSOM.Client

def start_on_demand(token, device_reference):
  body = {
    "cameraRef": device_reference
  }
  client = VSOM.Client.new("/camerarecording/startOnDemandRecording", body, token)
  return client.post_it()
~~~


Let's add the following to the top of our **example.py** file

~~~python
import VSOM.API.CameraRecording as CameraRecording
~~~

then add this to our **example.py** file, right above the ***vsom_run*** declaration

~~~python
def start_recording(token):
  camera_name      = ""
  device           = Camera.get_by_name(token, camera_name)
  device_reference = get_device_reference(device)
  res              = CameraRecording.start_on_demand(token, device_reference)
  return res
~~~

And let's modify our ***vsom_run*** method so that it looks like this

~~~python
def vsom_run():
  token = vsom_token()
  res   = start_recording(token)
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

The output should be similiar to [camera-start-recording.txt](assets/camera-start-recording.txt)

###Retrieve a snapshot of a recording at a specific time
Now that the camera has begun recording, we can retrieve a snapshot of a recording. In order to retrieve a snapshot, there will be 4 API calls that we will have to make.

1. /camera/getCameras
  * We will use this API endpoint to retrieve a camera reference so we can pass it to the next API call
2. /camerarecording/getRecordingCatalogEntries
  * We will use this API endpoint to retrieve a list of all recordings that a specific camera has stored on the media server
3. /camerarecording/getFirstLastForRecordingCatalogEntry
  * We will use this API endpoint to retrieve a recording catalog entry along with unix timestamps of the first and last frame of a given recording and the catalog id. We can then use this recording catalog id and the frame details to take a snapshot of the recording.
4. /camerarecording/getThumbnails
  * We will use this API endpoint to retrieve a snapshot of a recording


Add these three methods to our **CameraRecording.py** file

~~~python
def get_camera_recording_catalog_entries(token, filter):
  body = {
    "filter": filter
  }
  client = VSOM.Client.new("/camerarecording/getRecordingCatalogEntries", body, token)
  return client.post_it()

def get_first_last_recording_catalog_entry(token, device_reference, id):
  body = {
    "cameraRef": device_reference,
    "recordingCatalogEntryId": id
  }
  client = VSOM.Client.new("/camerarecording/getFirstLastForRecordingCatalogEntry", body, token)
  return client.post_it()

def get_thumbnails(token, camera_reference, id, start_time, end_time):
  body = {
    "request": {
      "cameraRef": camera_reference,
      "recordingCatalogEntryUid": id,
      "numThumbnails": 1,
      "startTimeInMSec": start_time,
      "endTimeInMSec": end_time,
      "forRecordings": False,
      "encoding": "JPG",
      "thumbnailResolution": "full",
      "thumbnailQuality": "high",
      "tokenExpiryInSecs": 300
    }
  }
  client = VSOM.Client.new("/camerarecording/getThumbnails", body, token)
  return client.post()
~~~

Add this to our **example.py** file, right above the ***vsom_run*** declaration

~~~python
def get_snapshot(token):
  camera_name      = ""
  device           = Camera.get_by_name(token, camera_name)
  device_reference = get_device_reference(device)
  filter = {
    "byCameraAlternateId": device["data"]["items"][0]["alternateId"] 
  }
  res = CameraRecording.get_camera_recording_catalog_entries(token, filter)
  id  = res["data"]["items"][0]["uid"]
  res = CameraRecording.get_first_last_recording_catalog_entry(token, device_reference, id)
  id  = res["data"]["uid"]
  current_time = int(time.time())
  last_frame   = res["data"]["lastFrame"]
  start_frame  = last_frame - 1000
  res          = CameraRecording.get_thumbnails(token, device_reference, id, start_frame, last_frame)
  return res
~~~

And let's modify our ***vsom_run*** method so that it looks like this

~~~python
def vsom_run():
  token = vsom_token()
  res   = get_snapshot(token)
  name  = int(time.time())
  name  = str(name) + '.jpg'
  file  = open(name, 'wb')
  file.write(res.content)
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

If successful, you should of created a new file (in the same directory as our **example.py** file) that contains the snapshot.

###Tell a camera to stop recording

Storing video on a storage medium quickly takes up a lot of space, let's write some code so that we can tell a camera to stop recording.

Add this method to our **CameraRecording.py** file

~~~python
def stop_on_demand(token, device_reference):
  body = {
    "cameraRef": device_reference
  }
  client = VSOM.Client.new("/camerarecording/stopOnDemandRecording", body, token)
  return client.post_it()
~~~

then add this to our **example.py** file, right above the ***vsom_run*** declaration

~~~python
def stop_recording(token):
  camera_name      = ""
  device           = Camera.get_by_name(token, camera_name)
  device_reference = get_device_reference(device)
  res              = CameraRecording.stop_on_demand(token, device_reference)
  return res
~~~
And let's modify our ***vsom_run*** method so that it looks like this

~~~python
def vsom_run():
  token = vsom_token()
  res   = stop_recording(token)
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

The output should be similiar to [camera-stop-recording.txt](assets/camera-stop-recording.txt)
