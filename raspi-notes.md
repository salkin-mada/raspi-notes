
# notes for Skandinavisk SuperCollider Klubb meetup 14th October 2019 (rpi workshop)

## set up SuperCollider on a rpi3 or rpi4 (and possibly a rpi2)

and run it either as headless or with the full scide (but for now without help documents).

* get raspbian [here] (https://www.raspberrypi.org/downloads/raspbian/)
* get and install [balena etcher] (https://www.balena.io/etcher/) (used to flash rasbpian onto sc card)
* flash raspbian to SD card <https://www.etcher.io/>
* connect ethernet, screen, and keyboard
* open the terminal on the RPi and type...
* if no screen and keyboard is available connect via ssh
    -
* `sudo raspi-config`
  - Change User Password
  - enable SSH in Interfacing Options (if not already done or default)
  - optional...
  - also change hostname
  - also enable VNC (if needed)
  -  _finish and reboot_
* open the terminal again and continue with these commands...
* `sudo apt-get update`
* `sudo apt-get upgrade` #optional - takes a while
* `sudo apt-get dist-upgrade` #optional - takes a while
* `sudo apt-get install tmux xvfb vim` #optional - not needed if end goal is to just get SuperCollider up and running
* build SuperCollider from source using this [guide](https://supercollider.github.io/development/building-raspberrypi.html)
    - until resolved, SuperCollider IDE needs to be built without qt-webengine (-DSC_USE_QTWEBENGINE:BOOL=OFF)
* or just follow the installation instructions for the SuperCollider standalone version: (Copied from redFrik's [repo](https://github.com/redFrik/supercolliderStandaloneRPI2#installation) )
* `sudo apt-get install qjackctl libqt5quick5 libqt5opengl5`
* `git clone https://github.com/redFrik/supercolliderStandaloneRPI2 --depth 1`
* `mkdir -p ~/.config/SuperCollider`
* `cp supercolliderStandaloneRPI2/sc_ide_conf_temp.yaml ~/.config/SuperCollider/sc_ide_conf.yaml`

NOTE: the last command will create a global sc_ide preference file from a template. At the moment SuperCollider IDE can not use a local configuration file, but hopefully this will change in the future. Also note that if you cloned or moved this repository somewhere else than in your home directory you should edit the yaml file with `nano ~/.config/SuperCollider/sc_ide_conf.yaml` to make the paths in there point to your standalone directory.

startup
--

To run the full IDE first open a terminal window and type...

* `qjackctl`

Select the correct soundcard (under setup/interfaces) and then start jackd. _(if usb soundcard is used also set periods to 3)_

Then open another terminal window and type...

* `cd supercolliderStandaloneRPI2`
* `export PATH=.:$PATH`
* `scide`

or simply just double click the desktop icon. SuperCollider IDE should start and run like normal - with scope, meter, plot, gui, animation, quarks etc.

The startupfile is located in the subdirectory `share/user/` and extensions you can put in `share/user/Extensions/` (first create that directory if it does not exist).

KNOWN ISSUES:

* 'libEGL warning: DRI2: failed to authenticate' that is posted in terminal at scide startup is harmless
* 'Open startup file' and 'Open user support directory' menu selections do not open the right file/folder.

jack
--

If you start SuperCollider without having Jack already running (like when autostarting or running headless), Jack will automatically launch when you boot the server. The audio settings then used are found in the file...

* `nano ~/.jackdrc`

_(this file is created by qjackctl so if you never ran qjackctl you might need to create this file manually.)_

The recommended jack audio settings are...

* `/usr/bin/jackd -P75 -dalsa -dhw:0 -p1024 -n3 -s -r44100`

and to set up Jack to use an external usb sound card change `-dhw:0` to `-dhw:1` like this...

* `/usr/bin/jackd -P75 -dalsa -dhw:1 -p1024 -n3 -s -r44100`

NOTE: the internal soundcard volume is by default set low (40). type `alsamixer` in terminal and adjust the pcm volume to 85 with the arrow keys, esc key exits.

autostart
--

* `sudo apt-get install xvfb`
* `crontab -e` #and add the following line to the end
  * `@reboot cd /home/pi/supercolliderStandaloneRPI2 && xvfb-run ./autostart.sh`
* `sudo reboot` #and supercollider should automatically start after a while and play some beating sine tones.

Then edit the autostart script to load whichever file. By default it will load `mycode.scd`.

headless
--

To run sclang+scsynth only from ssh...

* `export DISPLAY=:0.0`
* `cd supercolliderStandaloneRPI2`
* `./sclang -a -l ~/supercolliderStandaloneRPI2/sclang.yaml`

NOTE: one can also specify a .scd file to load when starting sclang like this: `./sclang -a -l ~/supercolliderStandaloneRPI2/sclang.yaml mycode.scd`

- - -

## simple midi usage

connect your class compliant midi device and..

```supercollider
MIDIClient.init; // check sources and destinations
MIDIIn.connectAll; // quick'n dirty connect to MIDI controller, we will only be using one
MIDIdef.trace
MIDIdef.trace(false); // to stop tracing
```

then run the following code (may need to be ajusted due to the chosen midi device cc layout i.e. midi numbering)

```supercollider
(
MIDIIn.connectAll;

// create easy assible Ndef's holding the midi data from knobs and buttons/keys (cc and note on/off messages)
fork{
	80.collect({arg i, item; item = "cc_%".format(i+1).asSymbol;}).do({arg paramName, i;
		var path = "/midi/%".format(paramName).asSymbol;
		Ndef(path, 0.0).kr(1);
		MIDIdef.cc("cc_%Responder".format(paramName).asSymbol, {arg val, num, chan, src;
			Ndef(path).bus.set(val);
		},
		ccNum: 1+i,
		chan: 0
		);
	});
	
	40.collect({arg i, item; item = "button_%".format(i+1).asSymbol;}).do({arg paramName, i;
		var value;
		var path = "/midi/%".format(paramName).asSymbol;
		Ndef(path, 0.0).kr(1);
		
		
		MIDIdef.noteOn("noteOn_%Responder".format(paramName).asSymbol, {arg velocity, note, chan, src;
			value = 1;
			Ndef(path).bus.set(value);
		},
		noteNum: 48+i,
		chan: 0
		);
		
		MIDIdef.noteOff("noteOff_%Responder".format(paramName).asSymbol, {arg velocity, note, chan, src;
			value = 0;
			Ndef(path).bus.set(value);
			
		},
		noteNum: 48+i,
		chan: 0
		);
		
		
	});
};
)
```

and example of midi ndefs usage

```supercollider
(
Ndef(\blabla, {
	var sig, efx, sum;
	var detune = Ndef.kr('/midi/cc_1').linlin(0,127,-50,177).poll(label: 'detune');
	var density = Ndef.kr('/midi/cc_2').linlin(0,127,0.05,2.0).poll(label: 'density');
	var duration = Ndef.kr('/midi/cc_3').linlin(0,127,0.1,9.0).poll(label: 'duration');
	var amp = Ndef.kr('/midi/cc_5').linexp(0,127,1e-5,0.99).poll(label: 'amplitude');
	var switch = Ndef.kr('/midi/button_10').poll(label: 'switch');
	
	sig = SinGrain.ar(Dust.kr(density), duration, [111,222,333,444]+detune.lag(2), amp);
	efx = CombC.ar(sig, 1.0, delaytime: LFNoise1.kr(0.05).linlin(-1.0,1.0,0.5,1.0), decaytime: 12.0) * switch.lag(0.5);
	sum = (sig + efx);
	Limiter.ar(Splay.ar(sum, 0.33, 0.55, LFNoise1.kr(0.1)), 0.99);
}).play;
)

Ndef(\blabla).free; // stop audio and post window info
```
_notice that if cmd+. or ctrl+. is used to stop the sound, then we need to rebuild the responders and kontrol rate ndefs above_


# more general notes

## headless notes
* useful
  - `ip a` (linux) -> (ifcondig is deprecated)
  - `ip addr show` (linux)
  - `ifconfig` (macOS)
  - `ipconfig` (windows)
  - `nmap -sn 1.2.3.0/24`
  - `nmap -sP 1.2.3.217`
  - `nmap -R 1.2.3.217` see if tcp ports for ssh or vnc is open
  _ `ssh pi@my-raspi` (pi@1.2.3.217) (username@1.2.3.217)


## how to solve problem with mounting a usb drive:

* `lsusb` - should do the trick
* if not then ..
* mounting usb
  - `ls /dev/`
  - `sudo mkdir /media/usb`
  - `sudo mount -t vfat -o uid=pi,gid=pi /dev/sda1 /media/usb`
  - `ls /media/usb`

## shutdown for raspberry pi

to have a button turn off the rpi

  * save this python script as 'shutdown.py' in /home/pi
  ```python
  #!/bin/python
  import RPi.GPIO as GPIO
  import os
  pin= 3
  GPIO.setmode(GPIO.BCM)
  GPIO.setup(pin, GPIO.IN)
  try:
      GPIO.wait_for_edge(pin, GPIO.FALLING)
      os.system("sudo halt -p")
  except:
      pass
  GPIO.cleanup()
  ```

* then edit crontab
  - `crontab -e` #and add the following...
  - `@reboot python /home/pi/shutdown.py`

to have the rpi turn off from a network command
  * add this somewhere to your rpi project code
  ```
  OSCFunc({|msg| msg.postln; "sudo halt -p".unixCmd}, \shutdown, recvPort: 52705);
  ```
* use this message to turn off your rpi via SuperCollider
  ```
  NetAddr("ip.of.your.rpi", 52705).sendMsg(\shutdown);
  ```
  * or use a broadcast message like this to turn off all rpis on the network
  ```
  NetAddr("192.168.1.255", 52705).sendMsg(\shutdown);
  ```

## static ip wlan0

note this is for raspbian jessie (and possibly newer buster - not tested) (things work differently on older versions than jessie)

* ssh to a raspberry and edit this file, adding the lines below
  - `sudo nano /etc/wpa_supplicant/wpa_supplicant.conf`
    ```
    network={
            ssid="myssid"
            psk="mypassword"
    }
    ```
* then edit this file, adding the relevant ip addresses...
  - `sudo nano /etc/dhcpcd.conf`
    ```
    interface wlan0
    static ip_address=192.168.1.150/24
    static routers=192.168.1.1
    static domain_name_servers=192.168.1.1 8.8.8.8
    ```


## backup the sd card

insert the sd card into a mac osx laptop and type

`diskutil list`

verify that your sd card is mounted at /dev/disk2 and then do...

`sudo dd bs=4m if=/dev/rdisk2 of=/Users/asdf/Desktop/ting2backup20170525.img`

with a 16gb card this process takes ~4.5min and it can easily be restored with <http://etcher.io>


## keep sc alive script

an example of how to set up a python script that monitor and restart supercollider if it stop sending osc (if it crashed)

* install pyosc if needed
  - `pip install pyosc --pre`
* make the below script start at bootup
  - `@reboot cd /home/pi/supercolliderStandaloneRPI2 && python surveillance.py`
* save the following python code as '/home/pi/supercolliderStandaloneRPI2/surveillance.py'

```python
#!/usr/bin/env python

#a script for checking if sc is alive
#send it an osc message once every x second
#else sc + jack will be forcefully restarted

#first install pyosc with this command...
#   pip install pyosc --pre

import subprocess
from time import sleep
from threading import Thread
from OSC import OSCServer

TIMEOUT= 30     #in seconds
OSCPORT= 50005  #network port

def oscInput(addr, tags, stuff, source):
       global received
       received= True
def timeout(): #called when no osc message received for x seconds
       subprocess.call(['pkill', 'sclang'])
       subprocess.call(['pkill', 'scsynth'])
       subprocess.call(['pkill', 'jackd'])
       p= subprocess.Popen(['xvfb-run', './autostart.sh'])

server= OSCServer(('0.0.0.0', OSCPORT)) #receive from everywhere on port x
server.addDefaultHandlers() #for dealing with unmatched messages
server.addMsgHandler('/alive', oscInput)
server_thread= Thread(target= server.serve_forever)
server_thread.start()
print server
received= True

try:
       while received:
               received= False
               sleep(TIMEOUT)
               if not received:
                       print 'timeout!!!!'
                       timeout()
                       received= True
except KeyboardInterrupt:
       print 'closing'
server.close()
server_thread.join()
print 'done'
```

then from sc do something like this to send a keep alive message from sclang

```supercollider
Routine.run({
	inf.do{
		NetAddr("127.0.0.1", 50005).sendMsg(\alive);
		25.wait;  //25 seconds
	};
});
```

or let scsynth send the keep alive message via sclang like this

```supercollider
(
SynthDef(\alive, {var dur= 25; SendReply.kr(Impulse.kr(1/dur), '/alive', 1)}).play;  //dur in seconds
OSCdef(\alive, {|msg| NetAddr("127.0.0.1", 50005).sendMsg(\alive)}, \alive);
)
```

## further rememberance / info

### build and include sc3-plugins on raspberry pi
* if not already done, install cmake
  - `sudo apt-get update && sudo apt-get upgrade`
  - then `sudo apt-get install cmake`
* see <https://github.com/redFrik/supercolliderStandaloneRPI2/blob/master/BUILDING_NOTES.md>
  - `git clone --recursive git://github.com/supercollider/supercollider --depth 1`
  - `git clone --recursive https://github.com/supercollider/sc3-plugins.git --depth 1`
  - `cd sc3-plugins`
  - `mkdir build && cd build`
  - `export CC=/usr/bin/gcc-4.8`
  - `export CXX=/usr/bin/g++-4.8`
  - `cmake -L -DCMAKE_BUILD_TYPE="Release" -DCMAKE_C_FLAGS="-march=armv7-a -mtune=cortex-a8 -mfloat-abi=hard -mfpu=neon"`
  - `-DCMAKE_CXX_FLAGS="-march=armv7-a -mtune=cortex-a8 -mfloat-abi=hard -mfpu=neon" -DSC_PATH=../../supercollider/`
  - `-DCMAKE_INSTALL_PREFIX=~/supercolliderStandaloneRPI2/share/user/Extensions/sc3-plugins ..`
  - `make -j 4` leave out flag ~~-j 4~~ on single core rpi models _(zero,1,2)_
  - `sudo make install`
  - `cd ~/supercolliderStandaloneRPI2/share/system/Extensions/`
  - `sudo chown -R pi SC3plugins`
  - `sudo chgrp -R pi SC3plugins`
  - `mkdir SC3plugins/bin`
  - `mv SC3plugins/lib/SuperCollider/plugins/*.so SC3plugins/bin/`
  - `mv SC3plugins/share/SuperCollider/Extensions/SC3plugins/* SC3plugins/`
  - `rm -rf SC3plugins/lib`
  - `rm -rf SC3plugins/share`
  - `rm -rf SC3plugins/local`

