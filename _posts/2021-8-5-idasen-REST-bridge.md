---
layout: post
title: Control your IKEA desk via PC
tags: Python Flask Swagger SmartHome REST RaspberryPi
comments: true
---

Recently I purchased a height-adjustable desk from IKEA called [IDÅSEN](https://www.ikea.com/ch/en/p/idasen-desk-sit-stand-brown-dark-grey-s39281004/) for my home office.  
Interestingly it comes with an app for your phone that lets you, via bluetooth, control the height of the desk. The app is the only way to have *presets* to move the desk to the proper heights for your standing/sitting positions once you found what fits for you.  
However I did not want to rely on my phone all the time and set out to find a different way. In this post I explain how I managed to use a [Raspberry Pi Zero](https://www.raspberrypi.org/products/raspberry-pi-zero/) to act as a bridge between any device in the network and the desk and allow to remote-control it without the need for any bluetooth.  

The final result can be found on [github](https://github.com/huserben/idasen-rest-bridge).

# Control the Desk without App
First I wanted to figure out whether it's even possible to interface with the desk without the app. A quick online search later revealed that it's not only possible, but quite some people are already doing it and offering packages.  

I liked [idasen](https://pypi.org/project/idasen/) python package that can be installed via pip. It seemed simple and had a [git repo](https://github.com/newAM/idasen) that was decently documented, so I gave it a try.

Using my laptop that has builtin bluetooth I paired with the Desk and then got it's MAC address via the properties - we'll be needing this in a bit.
> If you install the [official app](https://play.google.com/store/apps/details?id=com.linak.deskcontrol&hl=en&gl=US) you can rename your desk, so it's easier to find with other devices later on for manual pairing.

Then I installed idasen via `python -m pip install --upgrade idasen`  and initialized the configuration file by running `idasen init`.

This will create a config file under *.config/idasen/idasen.yaml* in the user directory with some default values:

``` bash
PS C:\Users\benja> idasen init
Failed to discover desk's MAC address
Created new configuration file at: C:\Users\benja\.config\idasen\idasen.yaml
PS C:\Users\benja> cat ~/.config/idasen/idasen.yaml
# https://idasen.readthedocs.io/en/latest/index.html#configuration
mac_address: AA:AA:AA:AA:AA:AA
positions:
  sit: 0.75
  stand: 1.1
```

**Note:** You can see that it tried to discover the desks MAC address. For that always failed which is why I had to adjust it manually. For this just modify the file and replace the default MAC address with the real one. 

Once you did this you can verify via `idasen height` whether it works - if you get the actual desk height back it works fine. We're almost ready to write some script now, but let's just find the proper position to move to.

## Setup your Height
My use case is that I want to switch between sitting and standing position rather frequently and I want the desk to move to the right height automatically. The idasen package offers this by allowing to move to predefined positions, by default we'll have *sit* and *stand* as you can see in the default idasen file above.  

So first I configured my preferred height for both sitting and standing and updated the file with that information. You can do it manually by adjusting the file after getting back the current height from idasen: `idasen height`.  
Or you make use of the built in functions for this. Adjust your desk to the height you wish and then execute `idasen save sit` or `idasen save stand` respectively. This will save the current desk height for the position specified.

Do your back a favor and check what proper ergonomic positions are, e.g. with [this guide](https://ergonofis.com/blogs/news/how-to-use-my-sitstand-desk-correctly).

## Write a Basic Script
Ok, scripting time. My goal is to click once and have the desk move to the position I want. I figured I just need to toggle the state. If it's in sitting position now I want it to move to standing and vice versa.  

For this I decided to check the current desk height, and if it's heigher than 1 metre I'll move it to the sitting position. Otherwise I'm probably already sitting and move it to standing position. I used *Popen* and *PIPE* from the *subprocess* module to invoke the *idasen* commands and parse the output (remember I need to get the current desk height). A 20 line script did the job perfectly:

``` python
#!/usr/bin/python

import subprocess
from subprocess import Popen, PIPE

process = Popen(["idasen", "height"], stdout=PIPE)
(output, err) = process.communicate()
exit_code = process.wait()

current_height = float(output.decode("utf-8").split(' ')[0])
print('Current Desk Height: ' + str(current_height) + 'm')

if current_height > 1.0:
    print("Desk is up, will move to sitting position")
    subprocess.run(["idasen", "sit"])
else:
    print("Desk is down, will move to standing position")
    subprocess.run(["idasen", "stand"])

print("Adjusted Desk Height:")
subprocess.run(["idasen", "height"])
```

Ok cool, that solves my one-click requirement. However I'm still running this on my laptop, which I do not really want to have on just for this, as I have a good old desktop pc...so let's see if we can use a Raspberry Pi for the job.

# Running it on a Raspberry Pi Zero
The Raspberry Pi Zero is an even smaller version of the regular Raspberry. It connects to 2.4 Ghz Wifis and has built in Bluetooth. Sounds good for our use case, let's try to run the script on it.

## Installing Python 3.8 or higher
First problem that arose was that Python is only available in version 3.7 for the Raspberry at the time of writing, however the idasen package needs at least 3.8. No problem, we just need a bit of time so we can build it ourselves on our little Pi.  

**Note:** The following operations might take a while (1-2h), a Raspberry Pi Zero is not the fastest machine out there...

First let's install some tools to make sure we can build everything:  
``` bash
sudo apt-get install -y build-essential tk-dev libncurses5-dev libncursesw5-dev libreadline6-dev libdb5.3-dev libgdbm-dev libsqlite3-dev libssl-dev libbz2-dev libexpat1-dev liblzma-dev zlib1g-dev libffi-dev tar wget vim
```

Then we can download the latest Python package (3.9.6 at the time of writing) and extract it:  
``` bash
wget https://www.python.org/ftp/python/3.9.6/Python-3.9.6.tgz
sudo tar zxf Python-3.9.6.tgz
```

Once we have it locally and extracted, let's build and install it:  
``` bash
cd Python-3.9.6
./configure --enable-optimizations
make
sudo make install
```  

Once this completed successfully run `python3` and you should see the proper version installed:  

``` bash
pi@raspberrypi:~ $ python3
Python 3.9.6 (default, Jul 21 2021, 22:03:42)
[GCC 8.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
```

## Pair Raspberry with Desk
Next we got to pair the desk with the Raspberry.

I faced quite some troubles with this, what helped was to make sure I do not run my phone with the app/bluetooth on. You might even want to unpair your phone or any other device you have already paired before continuing, as I had the feeling it was behaving strangely as long as I had multiple paired devices with bluetooth on.  

In order to pair it, we use *bluetoothctl*: `sudo bluetoothctl`. Then we need to register an agent: `agent on` followed by `default-agent`. After this we're ready to scan for devices. Put your desk in pairing mode and run `scan on` - you should start seeing all the devices that are found with their address. If you already know the device address (from the previous step) you can also skip the scanning. Once you know the address, run `pair XX:XX:XX:XX:XX:XX`.

You might have to type in `yes` to allow the pairing. After it was done successfully, you should see that your device is paired:

``` bash
[bluetooth]# paired-devices
Device FD:11:33:7C:BA:B4 Pablo Deskobar
[bluetooth]#
```

You can leave *bluetoothctl* and you're ready to go.

## Run the Script via Raspberry Pi
Copy over the idasen config file and the python script to your pi. Then you can install idasen via pip and run `idasen height`. Same as before, if you get the desk height, everything is working smoothly.

I had issues with this unless I ran it with sudo, as it would not find the bluetooth device. So if it fails you might want to try `sudo idasen height`.

Once you've got the connection setup, you can run the python script and toggle the desk height.  
Nice, now we don't need our bulky laptop anymore to control the desk, but only some tiny computer. But how can we trigger it now from any other device in the network? It might be ok to run it via ssh from my desktop, but it's not really nice. So let's make this better.

# Writing a REST Wrapper
Wouldn't it be nice if we could have some easily accessible, flexible and nicely documented API that almost any device could use? REST to the res(t)cue!  

The goal was mainly to just wrap all the commands the existing idasen package offers and expose them via REST. For this I chose [flask](https://flask.palletsprojects.com/en/2.0.x/) and the flask_restplus package to create a nice [Swagger](https://swagger.io/) page.

## Create Scaffolding

So first we want to create the Flask application:

``` python
from flask import Flask, request, abort
from flask_restplus import Api, Resource

flask_app = Flask(__name__)
app = Api(app = flask_app)

name_space = app.namespace('', description='idasen API')

if __name__ == '__main__':
    flask_app.run(debug=True, host='0.0.0.0')
```

If you run this, it will create a local server that listens on port 5000. If you browse there it will display all defined REST operations in a nice Swagger UI with the possibilities to try them out directly. If we run this we get the following:  

``` bash
 * Serving Flask app "main" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 288-991-003
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
127.0.0.1 - - [05/Aug/2021 17:00:57] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [05/Aug/2021 17:00:57] "GET /swaggerui/droid-sans.css HTTP/1.1" 200 -
127.0.0.1 - - [05/Aug/2021 17:00:57] "GET /swaggerui/swagger-ui.css HTTP/1.1" 200 -
127.0.0.1 - - [05/Aug/2021 17:00:57] "GET /swaggerui/swagger-ui-standalone-preset.js HTTP/1.1" 200 -
127.0.0.1 - - [05/Aug/2021 17:00:57] "GET /swaggerui/swagger-ui-bundle.js HTTP/1.1" 200 -
127.0.0.1 - - [05/Aug/2021 17:00:57] "GET /swaggerui/favicon-16x16.png HTTP/1.1" 200 -
127.0.0.1 - - [05/Aug/2021 17:00:57] "GET /swagger.json HTTP/1.1" 200 -
```

![Empty Swagger UI]({{ site.baseurl }}/images/posts/idasen_rest_bridge/swagger_empty.png)

## Add REST methods
Ok let's add some REST methods. Every REST *Resource* we want is defined in its own class. In there we can have the regular methods (*GET*, *POST*, *DELETE* and *PUT*). We can also annotate it with some comments to make a nice Swagger UI.

Let's start by wrapping the `idasen height` method, so we can send a *GET* and receive the height of the desk. This will look like this:  

``` python
def get_desk_height():
    print("Getting current desk height...")
    output, _, _ = run_idasen_command(["height"])
    desk_height = output.split(' ')[0]

    print(desk_height)
    return float(desk_height)

from subprocess import Popen, PIPE
def run_idasen_command(command_arguments, wait_for_exit = True):
    command = ["idasen"]
    command.extend(command_arguments)

    process = Popen(command, stdout=PIPE)

    if (wait_for_exit):        
        (output, err) = process.communicate()
        exit_code = process.wait()
        return output.decode("utf-8"), exit_code, err

@name_space.route("/height", methods=['GET', 'POST'])
class Height(Resource):
    @name_space.doc(responses={200: "The current height of the desk"})
    def get(self):
        return str(get_desk_height()), 200
```

This will make the method available under */height* and will return the desk height when a *GET* request is sent, together with a [200 response code](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status#successful_responses).  

The same way we can also add our *toggle* from the seperate script before:  

``` python
from idasen import cli

def position_exists(position_name):
    config = cli.load_config()
    return position_name in config["positions"]

def get_position_height(position_name):
    config = cli.load_config()
    return config["positions"][position_name]

def move_desk_to_position(position_name):
    print("Moving to height {0}m for {1} position".format(get_position_height(position_name), position_name))
    run_idasen_command([position_name], False)

@name_space.route("/toggle", methods=['POST'])
class Toggle(Resource):
    @name_space.doc(
        params={'sit_position': "The sitting position for toggling", "stand_position": "The standing position for toggling"},
        responses={
            202: "Starts moving desk to new position. If it's currently above 1m it will move to sitting position. Otherwise it will move to standing position",
            404: "Specified position for toggling does not exist"
        })
    def post(self):
        sit_position = request.args.get('sit_position', default="sit")
        stand_position = request.args.get('stand_position', default="stand")

        print("Toggling Desk Position between {0} and {1}".format(sit_position, stand_position))

        if (not position_exists(sit_position)):
            return "Position {0} does not exist".format(sit_position), 404
        elif (not position_exists(stand_position)):
            return "Position {0} does not exist".format(stand_position), 404

        current_height = get_desk_height()

        if current_height > 1.0:
            move_desk_to_position(sit_position)
            return "Moving Desk t {0}m".format(get_position_height(sit_position)), 202
        else:
            move_desk_to_position(stand_position)
            return "Moving Desk t {0}m".format(get_position_height(stand_position)), 202
```

Let's run this and see how our page now looks like:  

![Swagger UI]({{ site.baseurl }}/images/posts/idasen_rest_bridge/swagger_demo.png)

You can see that we have our two methods now available, together with a *Try it out* button that will allow us to specify parameters if needed and to send a request - perfect for testing! Also you can see that we get all the methods and possible return codes nicely visualized in the UI.

![Swagger Send Request]({{ site.baseurl }}/images/posts/idasen_rest_bridge/swagger_send_request.png)

And that's how we can wrap all the methods from the original idasen cli and expose REST methods. It is a bit of typing involved, but luckily I've got something prepared already...

## Running the whole thing
You can find the full file on [github](https://github.com/huserben/idasen-rest-bridge/blob/main/main.py). I advise to just clone the repo, install the dependencies and run it like this:  

``` python
python -m pip install --no-cache-dir -r requirements.txt
python main.py
```

It should then look like this:

![Complete Swagger UI]({{ site.baseurl }}/images/posts/idasen_rest_bridge/swagger_complete.png)

And you can do everything now via plain REST commands or via the Swagger UI. You have the flexibility to write your custom scripts in any language and on any platform. Feel free to write an app if you feel like it - just let me know once you're done ;-)

# Shell Commands
Lastly I want to show how I now trigger the desk from my desktop pc. Via swagger you could see the *curl* command that was sent to the server. I copy & pasted it and put it in a batch file:  

![Toggle Desk Batch]({{ site.baseurl }}/images/posts/idasen_rest_bridge/toggle_desk.png)

That's it, nothing else needed to execute it.

## Pinning it to the Windows Taskbar
However there was one remaining thing left. I really just want to click once. For this I want the script to be pinned to the taskbar. A *.bat* file is not attachable to the taskbar directly...however a shortcut is.  

You can create a new shortcut that executes `cmd.exe /c "D:\Scripts\Desk\ToggleDesk.bat"` - it will run cmd.exe and execute the batch file we've just created.  

If you want to go super fancy, you then can also change the icon of the short and lastly pin it to the Taskbar. One click for toggling the desk height!

![Shortcut]({{ site.baseurl }}/images/posts/idasen_rest_bridge/shortcut.png)

# Conclusion
In this post we learned how to interface with an IKEA desk via the *idasen* python package. We saw how to run it on a Raspberry Pi Zero and how we can wrap it in a REST interface.  
If you want to use yourself (on a raspberry pi or anywhere else where you can run Python) you can just clone the [git repo](https://github.com/huserben/idasen-rest-bridge) and run it as described in the read me.