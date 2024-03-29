# MindLamp
# AHI_mindLAMP
This repository is dedicated to establishing a Test Environment for mindLAMP as part of the Applied Health Informatics Program. 

The LAMP Platform is a solution for neuropsychiatric research and clinical care management. It was specifically created to streamline the administration of resarch sutdies and digitally-enabled clinics.
It uses numerous components to automate workflows, offering a simplified approach to clinical and data management. 

The patients or users uses the mindLAMP app, which can be downloaded on a mobile device on the AppStore whether it be an iOS or Android device. They will take part in taking surveys, playing cognitive games, or doing
other activites to collect active data. The app also collects passive data through Sensor events. This includes the mobile device's accelerometer, GPS, pedometer, and other background capabilities. 

Clinicians and Researchers can use a dashboard to create, customize, and schedule activities for the patients to interact with. The dashboard has a fully functioning Data Portal for a better view and extraction function for the data. 

## Setting up the Server
- Modified the docker-compose to include traefik in it
```
Docker Stack: **traefik.yml**
```
- Modified to remove swarm, this is just for testing purposes

## How to Deploy
Create a new network called public to connect all externally accessible services.
```
docker network create --driver overlay --attachable public
```

Using your DNS provider of choice, provision a domain name (here we use [example.com](http://example.com) to represent your domain name and 1.1.1.1 to represent your node's IP address).

Replace administrator@example.com with your configuration variable. 
```
version: "3.7"
services:
  traefik:
    image: traefik:latest
    command:
      - "--log.level=INFO"
      - "--accesslog=true"
      - "--api=true"
      - "--providers.docker=true"
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.exposedByDefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.websecure.http.tls.certResolver=default"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      - "--entrypoints.web.http.redirections.entryPoint.permanent=true"
      - "--certificatesResolvers.default.acme.email=administrator@example.com"
      - "--certificatesResolvers.default.acme.storage=/data/acme.json"
      - "--certificatesResolvers.default.acme.tlsChallenge=true"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "traefik-ssl:/data/"
    ports:
      - target: 80
        protocol: tcp
        published: 80
        mode: ingress
      - target: 443
        protocol: tcp
        published: 443
        mode: ingress
    networks:
      - public
    deploy:
      mode: replicated
      placement:
        constraints:
          - node.role == manager
networks:
  public:
    external: true
volumes:
  traefik-ssl:
```
```
docker stack deploy --compose-file traefik.yml router
```

Create a /data folder in the node that will be hosting the database. 
```
mkdir -p /data/db
```

```
openssl rand -hex 8 # DB_PASSWORD_HERE
openssl rand -hex 32 # ROOT_ENCRYPTION_KEY_HERE
```
- When it first runs, it will generate an admin password that you will be able to see in the terminal, it will look something like:
```
server_1         |       ┌────────────────────────┬────────────────────────────────────────────────────────────────────┐
server_1         |       │        (index)         │                               Values                               │
server_1         |       ├────────────────────────┼────────────────────────────────────────────────────────────────────┤
server_1         |       │ Administrator Password │ '34b8b1................f6cd' │
server_1         |       └────────────────────────┴────────────────────────────────────────────────────────────────────┘

```

To self host the LAMP dashboard, replace this variables with your custom configuration variables.

* dashboard.example.com [The address you will use to access the LAMP dashboard. (If deploying the dashboard)]
* ROOT_ENCRYPTION_KEY_HERE 
* DB_PASSSWORD_HERE 
* YOUR_PUSH_KEY_HERE 
* api.example.com 

```
version: '3.7'
services:
  dashboard:
    image: ghcr.io/bidmcdigitalpsychiatry/lamp-dashboard:2023
    logging:
        driver: "json-file"
        options:
          max-size: "50m"
    environment:
      REACT_APP_LAMP_RESEARCHER_ALIAS: 'Investigator'
    networks:
      - public
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:80 || exit 1
    deploy:
      mode: replicated
      update_config:
        order: start-first
        failure_action: rollback
      labels:
        portainer.autodeploy: 'true'
        traefik.enable: 'true'
        traefik.http.routers.lamp_dashboard.entryPoints: 'websecure'
        traefik.http.routers.lamp_dashboard.rule: 'Host(`dashboard.example.com`)'
        traefik.http.routers.lamp_dashboard.tls.certresolver: 'default'
        traefik.http.services.lamp_dashboard.loadbalancer.server.port: 80
      placement:
        constraints:
          - node.role == manager
networks:
  public:
    external: true
```

## How to Connect

- Example endpoint of how to connect: 
```python
import requests 

test_url = 'http://localhost:3000/api/v1/users/login'

test_data = {
    "username": "admin",
    "password": "34b8b1................f6cd"
}

```

Now run the dockerfile
```
docker stack deploy --compose-file lamp.yml lamp
```

## Logging in as as an Admin

Input your server address, username, and generated password.  
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/16.JPG)

## Creating an Investigator and your first User

Create an Investigator by clicking the add button. 
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/1.png)
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/2.JPG)

Create a user with a group. 
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/3.JPG)


## Setting up Surveys

Go to this [link](https://docs.lamp.digital/start_here/instruments) to download all the available surveys in JSON formatting. 

From the user or group you created, add the JSON files for the surveys here. 
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/5.png)

Add and import activities. Surveys are listed as activities. 
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/4.png)

## Setting up Activities / Games

Coding Activities / Games into mindLAMP requires knowing the name of the game and the parameters. See sample code below for more information.

Start by importing the requests packages.

Determine your endpoint.
```python
endpoint = 'https://mindlamp.api.{YOUR_ENDPOINT}.com/activity_spec'
```

Enter in the appropriate header information.
```python
headers = {
    'Content-Type': 'application/json',
    'Authorization': 'Basic {USER}:{PASSWORD},
    }
```

The following is sample code for the balloon risk game:
```python
bodytosend = {
    'name': 'lamp.balloon_risk',
   'balloon_count': 15,
   'breakpoint_mean': 64.5,
   'breakpoint_std': 37,
   'static_data': {'points': 55},
   'temporal_slices': [
       {'duration': 1821, 'item': 1, 'level': 1, 'type': True, 'value': 1},
       {'duration': 876, 'item': 2, 'level': 1, 'type': True, 'value': 1},
       {'duration': 425, 'item': 3, 'level': 1, 'type': True, 'value': 1},
       {'duration': 167, 'item': 8, 'level': 1, 'type': True, 'value': 1},
       # ... other entries ...
       {'duration': 151, 'item': 74, 'level': 2, 'type': True, 'value': 1},
       {'duration': 178, 'item': 75, 'level': 2, 'type': True, 'value': 1},
       {'duration': 161, 'item': 76, 'level': 2, 'type': False, 'value': 0},
       # ... other entries ...
       {'duration': 809, 'item': 1, 'level': 15, 'type': True, 'value': 1}
   ],
   'timestamp': 1642779164213,
   'duration': 78365,
   'activity': '8vw829aemgdxy8f1fvk9'
}
```
The of this activity is lamp.balloon_risk, the following information after can be adjusted.

Another example:

```python
bodytosend = {
    'name': "lamp.cats_and_dogs",
   'duration': 445841,
   'static_data': 
   {'StartTime': None, 'correct_answers': 22, 'point': 1, 'score': 49, 'type': 1, 'wrong_answers': 1},
   'temporal_slices': 
   [{'duration': 5243, 'item': 9, 'level': 1,'type': True,'value': None},
    {'duration': 2611, 'item': 5, 'level': 1, 'type': True, 'value': None},
    {'duration': 1125, 'item': 8, 'level': 1, 'type': True, 'value': None},
    {'duration': 2688, 'item': 2, 'level': 1, 'type': True, 'value': None},
    {'duration': 946, 'item': 3, 'level': 1, 'type': True, 'value': None},
    {'duration': 1000, 'item': 4, 'level': 1, 'type': True, 'value': None},
    {'duration': 4568, 'item': 4, 'level': 1, 'type': True, 'value': None},
    {'duration': 1035, 'item': 6, 'level': 1, 'type': True, 'value': None},
    {'duration': 843, 'item': 3, 'level': 1, 'type': True, 'value': None},
    {'duration': 798, 'item': 2, 'level': 1, 'type': True, 'value': None},
    {'duration': 4298, 'item': 1, 'level': 1, 'type': True, 'value': None},
    {'duration': 854, 'item': 8, 'level': 1, 'type': True, 'value': None},
    {'duration': 1397, 'item': 5, 'level': 1, 'type': True, 'value': None},
    {'duration': 885, 'item': 9, 'level': 1, 'type': True, 'value': None},
    {'duration': 4338, 'item': 10, 'level': 1, 'type': True, 'value': None},
    {'duration': 624, 'item': 9, 'level': 1, 'type': True, 'value': None},
    {'duration': 2846, 'item': 3, 'level': 1, 'type': True, 'value': None},
    {'duration': 2002, 'item': 8, 'level': 2, 'type': False, 'value': None},
    {'duration': 1981, 'item': 5, 'level': 2, 'type': True, 'value': None},
    {'duration': 731, 'item': 3, 'level': 2, 'type': True, 'value': None},
    {'duration': 4555, 'item': 4, 'level': 2, 'type': True, 'value': None},
    {'duration': 1608, 'item': 3, 'level': 2, 'type': True, 'value': None},
    {'duration': 910, 'item': 5, 'level': 2, 'type': True, 'value': None}],
   'timestamp': 1649447701749,
   'activity': 'sjfvrd7jpjyjzbkgwex4'
}
```
The name for this game is lamp.cats_and_dogs and note that there is the same timestamp and activity section but no duration for this section. Those last sections are considered optional according to the documentation.

From there you should push these games into the API.
```python
response = requests.post(endpoint, headers=headers, json=bodytosend)
response.status_code
response.text
```

And you're done!

More information can be found here: [link](https://github.com/BIDMCDigitalPsychiatry/LAMP-activities/tree/master) and [link](https://docs.lamp.digital/develop/build_new_activities)

## Setting up Sensors

Start by importing the package, requests. 

We are going to use an HTTP Post request to an endpoint to post sensors to the mindLAMP API. 

```python
import requests
```

Set endpoint equal to your endpoint. 
```python
endpoint = 'https://mindlamp.api.{YOUR_ENDPOINT}.com/sensor_spec'
```

Next, we include the headers in the HTTP request. The headers specify the content type to JSON and includes the authentication needed. 
```python
headers = {
    'Content-Type': 'application/json',
    'Authorization': 'Basic {USER}:{PASSWORD},
    }
```

Next, we include what is needed from the POST request. Replace "activity_recognition" with any of the Sensor types you need from this [link](https://docs.lamp.digital/using/sensors/). 
```python
bodytosend = {"name":"lamp.activity_recognition"}
```

Now, we create the HTTP POST request and print out the results. 
```python
response = requests.post(endpoint, headers=headers, json=bodytosend)
print(response.status_code)
print(response.text)
```

The Sensors should now be able to be added to a user/group from this screen. 
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/6.JPG)


## Creating additional Users/User Groups
Return this this screen to create a User or Group. 
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/3.JPG)

It's important to note to create multiple users if you have multiple, separate people using the application. When creating a new user, a new participant ID will be generated for each individual. 
This will be important later when we try to extract the data. When creating another User, you can duplicate the activities and sensors from another User. 

## Assigning Credentials
Go back to the user screen and click on the key icon to manage credentials. 
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/7.png)

Press the '+' icon to create additional credentials.  
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/8.JPG)

Fill in the information needed for the user.  
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/9.JPG)

After which, the user can log in with the newly created credentials. Each participant ID, meaning each user or user group you created, can have multiple credentials. 
Assigning multiple credentials however, does not create more users or user groups. Instead, create additional users or user groups and assign credentiaals to them as needed.

## User Perspective
After logging in with the given credentials, users should be able to see the activated activies. 
In order to track sensor data, the user will have to enable it on their mobile device. 
Sensors function as a passive data collection. For example, using the motion detected from a mobile device, we can collect accelerometer information. 

"Assess" contains different features to obtain active data. 


On the Assess screen, users will be able to play games or answer surveys. This is the desktop view.  
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/11.JPG)

Clicking on a survey, the user will be prompted to answer some questions.  
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/12.JPG)

## Extracting data using Python 
### Getting the Research ID
Start by installing the LAMP package and importing it.  
```python
pip install LAMP-core
import LAMP
```

As an administrator, you can see researcher IDs on the data portal screen. 
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/13.JPG)

Alternatively, you can perform a get request. 

Connect to the LAMP server 
```python
LAMP.connect("api.lamp.digital", "email@address.com", "password")
```

Make a GET request to the /researcher endpoint
```python
response = LAMP.Researcher.list()
```

### Getting the User ID. 
We can get each user's User ID by clicking on each user as a researcher. 
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/14.JPG)

Alternatively, after getting your researcher ID, we can get all the User IDs by one researcher. 

Set Test_researcher to a researcher ID. 
```python
TEST_RESEARCHER = 'z61npy2gv07hwpkrwzba'
```

Now run this line to receive all participant or user IDs by one researcher. 
```python
[LAMP.Participant.all_by_study(study['id'])['data'] for study in LAMP.Study.all_by_researcher(TEST_RESEARCHER)['data']]
```
The result should be all the user IDs. 

### Getting Activity Event Data
To see the data performed by a user, we will need a User ID. 

Set participant_1 equal to a user ID. 
```python
participant_1 = "U1679776962"
```

Next, run this line to see all activity events.
```python
LAMP.ActivityEvent.all_by_participant(participant_1)[0]
```
Through this method, we can see one activity performed by the user through indexes. We can change the [0] to [1] to see the next activity.
We can use the enumerate function to print all studies performed by a user through this method. 
```python
participant_1 = "U1679776962"
activity_events = LAMP.ActivityEvent.all_by_participant(participant_1)['data']

# Iterate over all activity events
for index, activity_event in enumerate(activity_events):
    print(f"Activity Event {index + 1}: {activity_event}")
```

We can also save the activities in a CSV file through this method. 
```python
import pandas as pd

participant_1 = "U1679776962"
activity_events = LAMP.ActivityEvent.all_by_participant(participant_1)['data']
events = LAMP.SensorEvent.all_by_participant(participant_1, origin='lamp.gps.contextual')['data']

data = []

for index, activity_event in enumerate(activity_events):
    print(f"Activity Event {index + 1}: {activity_event}")
    
    if 'temporal_slices' in activity_event:
        for slice_data in activity_event['temporal_slices']:
            data.append({
                'event_type': 'Mental Health Assessment',
                'timestamp': activity_event['timestamp'],
                'item': slice_data['item'],
                'value': slice_data['value'],
                'duration': slice_data['duration']
            })
    elif 'static_data' in activity_event:
        data.append({
            'event_type': 'Game or Task',
            'timestamp': activity_event['timestamp'],
            'score': activity_event['static_data']['score'],
            'total_attempts': activity_event['static_data']['total_attempts'],
        })

if len(data) > 0:
    df1 = pd.DataFrame(data)
    print(df1.head(20))
```

### Getting Sensor Event Data
We can also read sensor event data through this method.
```python
LAMP.SensorEvent.all_by_participant(participant_1)
```

### Alternate method using Requests
Alternatively, we can also view activity or sensor event data through requests. 

Set up the base_url
```python
base_url = 'https://mindlamp.api.yourendpoint.com/'
```

Set up your credentials
```python
user = 'user'
password = 'password'
```

Set up participant ID
```python
participant_id ='U1679776962'
```

Create an endpoint and a final URL. 
```python
endpoint = f"participant/{participant_id}/activity_event/"
final_url = base_url + endpoint
```

Make a GET request to the endpoint. 
```python
response = requests.get(final_url, auth=(email, password))
print("Status Code:", response.status_code)
print("Response Text:", response.text)
```

### Alternate Method using Dashboard
If you do not want to use Python, you can download a CSV file from this screen using the download feature underneath the researcher. 
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/13.JPG)
![](https://github.com/jas-tang/AHI_mindLAMP/blob/main/images/15.JPG)

## Issues found in mindLAMP Official Documentation

Posted code:
```python import LAMP
import pandas as pd 
LAMP.connect("MY_EMAIL_ADDRESS_HERE", "MY_PASSWORD_HERE")

for participant in LAMP.Participant.all_by_researcher("me")['data']:
    data = []
    events = LAMP.SensorEvent.all_by_participant(participant['id'], origin='lamp.gps.contextual')['data']
    for event in events:
        data.append({
            'timestamp': event['timestamp'],
            'UTC time': "",
            'latitude': event['data']['latitude'],
            'longitude': event['data']['longitude'],
            'altitude': 1.0,
            'accuracy': 1.0
        })
    # Don't make CSV files for participants without any `lamp.gps.contextual` events.
    if len(data) > 0:
        pd.DataFrame.from_dict(data, orient='columns').to_csv(f"{participant['id']}.csv", index=False)
```

This code was not able to pull down all the data. Also the data that was pulled did not include everything.

Fixed code:
```python
import pandas as pd

participant_1 = "U1679776962"
activity_events = LAMP.ActivityEvent.all_by_participant(participant_1)['data']
events = LAMP.SensorEvent.all_by_participant(participant_1, origin='lamp.gps.contextual')['data']

data = []

for index, activity_event in enumerate(activity_events):
    print(f"Activity Event {index + 1}: {activity_event}")
    
    if 'temporal_slices' in activity_event:
        for slice_data in activity_event['temporal_slices']:
            data.append({
                'event_type': 'Mental Health Assessment',
                'timestamp': activity_event['timestamp'],
                'item': slice_data['item'],
                'value': slice_data['value'],
                'duration': slice_data['duration']
            })
    elif 'static_data' in activity_event:
        data.append({
            'event_type': 'Game or Task',
            'timestamp': activity_event['timestamp'],
            'score': activity_event['static_data']['score'],
            'total_attempts': activity_event['static_data']['total_attempts'],
        })

if len(data) > 0:
    df1 = pd.DataFrame(data)
    print(df1.head(20))
```
This code also creates a dataframe for data analysis.

When setting up Sensors, a similar issue reoccurs in the official mindLAMP documentation. 
```
endpoint = 'https://mindlamp.api.{YOUR_ENDPOINT}.com/sensor_spec'
headers = {
    'Content-Type': 'application/json',
    'Authorization': 'Basic {USER}:{PASSWORD},
    }
bodytosend = {"name":"lamp.insert_here"}
response = requests.post(endpoint, headers=headers, json=bodytosend)
print(response.status_code)
print(response.text)
```
The official docmentation uses "ID" instead of "name", which causes an error. 
