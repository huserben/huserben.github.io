---
layout: post
title: Automating the Kingsmith WalkingPad
tags: Python Flask SmartHome REST RaspberryPi Postgres Grafana
comments: true
---

Inspired by [Peter Wynands post](https://peter-wynands.medium.com/1000-km-behind-my-desk-87ab44b4067c) about his first 1000km on a treadmill that he's using while working, I ended up buying one for my own.
After a few days of using it I got tired of having to sync via the Smartphone. The app was not always synching the data properly. There is no real export option for the data. And it's not even having any cloud sync option.
In the spirit of my [recent automation experience](https://huserben.github.io/idasen-REST-bridge/) I put on my coding hat and started building something better. The end result is that I can control my [Kingsmith WalkingPad A1](https://www.galaxus.ch/de/s3/product/kingsmith-walkingpad-a1-laufband-13320225) via Raspberry Pi, fetch the recent data from it, push it to a Postgres database hosted on Azure and visualize it in Grafana:

![Grafana Dashboard]({{ site.baseurl }}/images/posts/walkingpad_automation/grafana.png)

The code to control/access the WalkingPad is stored on [github](https://github.com/huserben/walkingpad) - Feel free to clone/fork/adjust to your needs.

# Goal
The end goal was that I could press one button on my desktop pc and it would fetch the data from the WalkingPad so that it is visualized on a Grafana dashboard. Because the WalkingPad connects via Bluetooth and my Desktop PC does not have Bluetooth, I wanted a REST API on a Raspberry Pi that is then communicating with the WalkingPad.

# Accessing the WalkingPad
First of all was the question how to communicate with the treadmill when not using the App. Luckily there are people smarter than me in the world that provide their code to the world. A quick [search on github](https://github.com/search?q=walkingpad) showed that there are some existing libraries. [ph4-walkingpad](https://github.com/ph4r05/ph4-walkingpad) from [Dušan Klinec](https://github.com/ph4r05) looked especially promising: It has a nice documentation and is written in python. Exactly what we need!

After a quick test locally I verified that indeed I can use this library to access my WalkingPad!
So let's write a REST API based on this library.

# Connecting to the WalkingPad
In order to do anything, you must connect to the WalkingPad. In order to connect to the WalkingPad you need to know the MAC Address of it. To figure this out just run the [scan.py script](https://github.com/huserben/walkingpad/blob/main/scan.py). This will scan for nearby devices. You should see a device named "WalkingPad":

![Scan]({{ site.baseurl }}/images/posts/walkingpad_automation/scan.jpg)

Once we have the address (here _57:4C:4E:27:23:DB_) we can put it in a yaml file with the following content:

``` yaml
address: 57:4C:4E:27:23:DB
```

This will then be read from the REST API during runtime.

# Changing Mode
As a first test I wanted to change the mode of the WalkingPad between _Manual_ and _Standby_. For creating the REST API I used [flask](https://flask.palletsprojects.com/en/2.0.x/async-await/). Important to know, as we use the ph4_walkingpad library which is using async commands, we must install flask with the async extra:

``` python
pip install flask[async]
```

Then we can create a method does the following:
1. Connect to WalkingPad
2. Set WalkingPad to desired Mode
3. Disconnect

This would translate to the following code:

``` python
from flask import Flask, request
from ph4_walkingpad import pad
from ph4_walkingpad.pad import WalkingPad, Controller
from ph4_walkingpad.utils import setup_logging
import asyncio
import yaml

app = Flask(__name__)

log = setup_logging()
pad.logger = log
ctler = Controller()

def load_config():
    with open("config.yaml", 'r') as stream:
        try:
            return yaml.safe_load(stream)
        except yaml.YAMLError as exc:
            print(exc)

async def connect():
    address = load_config()['address']
    print("Connecting to {0}".format(address))
    await ctler.run(address)
    await asyncio.sleep(1.0)


async def disconnect():
    await ctler.disconnect()
    await asyncio.sleep(1.0)

@app.route("/mode", methods=['POST'])
async def change_pad_mode():
    new_mode = request.args.get('new_mode')
    print("Got mode {0}".format(new_mode))

    if (new_mode.lower() == "standby"):
        pad_mode = WalkingPad.MODE_STANDBY
    elif (new_mode.lower() == "manual"):
        pad_mode = WalkingPad.MODE_MANUAL
    else:
        return "Mode {0} not supported".format(new_mode), 400

    try:
        await connect()

        await ctler.switch_mode(pad_mode)
        await asyncio.sleep(1.0)
    finally:
        await disconnect()
    
    return new_mode

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5678)
```

Note how we read the address during the connection from the yaml file in the *load_config* method.

Next up: Getting the data from my last walk.

# Getting Data
Controlling the mode is mildly impressive, but if we could get the data from my last walk that would be a step in the right direction. A bit of poking in the docs and code from the [ph4_walkingpad library](https://github.com/ph4r05/ph4-walkingpad) I found a way that works.  

We can register an event handler to get the latest record stored on the walking whenever we ask the controller for the history. This we can keep as dictionary and return to the caller when we ask for the history.

That results in following code additions:

``` python
last_status = {
    "steps": None,
    "distance": None,
    "time": None
}


def on_new_status(sender, record):

    distance_in_km = record.dist / 100
    print("Received Record:")
    print('Distance: {0}km'.format(distance_in_km))
    print('Time: {0} seconds'.format(record.time))
    print('Steps: {0}'.format(record.steps))

    last_status['steps'] = record.steps
    last_status['distance'] = distance_in_km
    last_status['time'] = record.time

@app.route("/history", methods=['GET'])
async def get_history():
    try:
        await connect()

        await ctler.ask_hist(0)
        await asyncio.sleep(1.0)
    finally:
        await disconnect()

    return last_status

ctler.handler_last_status = on_new_status
```

If we execute a `GET` request to `/history` we will get the last run:

![History]({{ site.baseurl }}/images/posts/walkingpad_automation/history.png)

Ok then, let's think about how to store this data.

# Storing Data in Database
I chose to use a Postgres database to store the data I get from my walk. You can easily host a database on any device as it's free and available for all platforms. I'm using an instance on my raspberry pi for testing purpose and an [instance hosted on Azure](https://azure.microsoft.com/en-us/services/postgresql/) for the real data.

## Table
The database consists of a single table I call *exercise*. In there it has 4 columns:
- day (date) with the current date (YYYY-MM-dd)
- steps (integer) that holds the number of steps
- duration (integer) with the time walked in minutes
- distance (real) with the distance walked in km

Once we have a table with those rows we can start storing data.

## Configuration
For having different database configurations we use the yaml based config file from above. We can extend it with a database entry and specify the properties we need for the connection:

``` yaml
database:
  host: 192.168.1.235
  port: 5432
  dbname: "exercise"
  user: dbUserName
  password: dbPassword
```

This allows us also to easily switch the db we're working against, for example if we create a lot of test data we might not want this in our production db.

## REST API
Ok everything is setup, we can add the code. For accessing the database we're using the psycop2 library. First we create the connection with the data from the config file. Then we calculate the current date and convert the duration from seconds to minutes before we insert a new entry.

This is triggered by a `POST` request to `/save`. From there we will fetch the last status that we have stored and save it in the db:

``` python
import psycopg2
from datetime import date


def store_in_db(steps, distance_in_km, duration_in_seconds):
    try:
        db_config = load_config()['database']
        conn = psycopg2.connect(host=db_config['host'], port=db_config['port'],
                                dbname=db_config['dbname'], user=db_config['user'], password=db_config['password'])
        cur = conn.cursor()

        date_today = date.today().strftime("%Y-%m-%d")
        duration = int(duration_in_seconds / 60)

        cur.execute("INSERT INTO exercise VALUES ('{0}', {1}, {2}, {3})".format(
            date_today, steps, duration, distance_in_km))
        conn.commit()

    finally:
        cur.close()
        conn.close()

app.route("/save", methods=['POST'])
def save():
    store_in_db(last_status['steps'], last_status['distance'], last_status['time'])

```

Once executed we can check whether we have the entry in our db that we expect:

![Postgres Entry]({{ site.baseurl }}/images/posts/walkingpad_automation/postgres.png)

# Making it more convenient
As I'm lazy and I wanted to only connect to the device once and execute all commands I created another endpoint that is putting it all together:
- Setting the mode to standby
- Getting the history data
- Storing it in the DB

``` python
@app.route("/finishwalk", methods=['POST'])
async def finish_walk():
    try:
        await connect()
        await ctler.switch_mode(WalkingPad.MODE_STANDBY)
        await asyncio.sleep(1.0)
        await ctler.ask_hist(0)
        await asyncio.sleep(1.0)
        store_in_db(last_status['steps'], last_status['distance'], last_status['time'])
    finally:
        await disconnect()

    return last_status
```

So now when I'm done walking I can send a single `POST` request to `/finishwalk` and within a few seconds my data will be saved in the db. Nice, now it just has to look nice.

# Visualizing the Data in Grafana
[Grafana](https://grafana.com/grafana/) is an open source solution for visualizing...*anything*. We have *something* to visualize, so it seemed like a good fit. Grafana, much like postgres, is usable on many different devices, including the raspberry pi. So after [installing it](https://grafana.com/docs/grafana/latest/installation/) you can browse to the Grafana Dashboard on port 3000 on the machine you installed it on.

There you can add your postgres db as a datasource:
![Grafana Datasource Config]({{ site.baseurl }}/images/posts/walkingpad_automation/datasource.png)

Now you can create a dashboard and add your widgets. For example counting the steps today:

``` sql
SELECT
  SUM(steps)
FROM exercise
Where day = CURRENT_DATE
```

or visualizing the distance walked in the last 30 days:

``` sql
SELECT
  SUM(distance)
FROM exercise
Where day > CURRENT_DATE - 30
  AND day <= CURRENT_DATE
```

Add whatever data you care about and create your own dashboard:
![Grafana Dashboard]({{ site.baseurl }}/images/posts/walkingpad_automation/grafana.png)

# Conclusion
With everything put together, I have a flexible visualization of my walking data without having to depend on an unstable app. My data is stored in a database that is in the cloud and I can simply click one button to put my WalkingPad into Standby mode and collect the latest data. Goal achieved.


*This post was created while walking