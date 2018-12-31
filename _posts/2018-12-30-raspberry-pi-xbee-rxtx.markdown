---
layout: post
title:  "Raspberry Pi RX/TX with XBee Devices"
date:   2018-12-30 16:30:00 -0400
categories: raspberry-pi raspbian xbee python linux electronics
---
Experimenting with electronics (and somewhat re-learning much of what I've forgotten since college), this post
details how to utilize [Raspberry Pi](https://www.raspberrypi.org/) controllers to communicate with each other
wirelessly using [XBee](https://www.digi.com/resources/documentation/digidocs/pdfs/90000982.pdf) Series 1 modules.
The learnings and circuit/communication methods lay the groundwork for more complicated robotics remote control
capabilities.

## Background

Setting up a simple transmitter/receiver pair of XBee-driven Raspberry Pi devices will be the basis for this
tutorial. In the interest of keeping it simple (not involving Analog to Digital/ADC devices given the Raspberry
Pi 3 does not have analog inputs), we will be using momentary push buttons for the directional control, and
LEDs to reflect the receiver end action being taken based on the controller.

The code for this project can be cloned from the [following repository](https://github.com/jekhokie/raspi-projects)
under the `11-xbee-send-receive` folder as a starting point.

## Parts List

In order to complete this tutorial, you'll need the following parts. Note you can probably take variants of
any of the below parts/components, but this tutorial was written with the following specific parts:

**Transmitter**:

- Raspberry Pi Model 3 B+
- MicroSD Card (size 16GB+ recommended)
- XBee Series 1 (S1)
- 4x push-buttons

**Receiver**:

- Raspberry Pi Model 3 B+
- MicroSD Card (size 16GB+ recommended)
- XBee Series 1 (S1)
- 4x LEDs
- 4x 270 Ohm Resistors

**Miscellaneous**:
- XCTU Software
- Wires
- 2x HDMI cables (one for each Raspberry Pi, or 1 total that you can switch between devices)
- Zip File of Raspbian Stretch (Rel. 2018-11-13, Kernel 4.14)
- Python 3 (on Raspbian OS)
- (Optional) Arduino with XBee shield (used to program the XBee devices)

Note that the Arduino listed above will be used to program the XBee devices in this tutorial (as I had a
couple laying around with the XBee shields associated, making it easy to program the XBee devices through
XCTU using the shields). You can also program the XBee using a discover shield or something similar that
can be hooked up to the computer running XCTU.

## Raspberry Pi Operating System

The Raspberry Pi requires an operating system to function (typically people will install the
[Raspbian](https://www.raspberrypi.org/downloads/raspbian/) operating system). First, download the Raspbian
OS you wish to use - for this tutorial, for compatibility, it's recommended you use the Raspbian Stretch
version (Released 2018-11-13, Kernel 4.14, or similar) in "with Desktop and recommended software" packaging.
Download the zip file and store somewhere you can find it.

Next, you will need to load the OS onto your Micro SD card. You can perform this by using the linux command
`dd` if you're familiar with it, but for simplicity, we'll use the [Etcher](https://www.balena.io/etcher/)
program. Download the Etcher program for your computer's operating system and launch it. The process is
pretty straightforward - select the microSD card (connected to your computer), select the OS image you wish
to load (the Raspbian image in zip format), and click "Flash". Several minutes later, you'll have a fully
functioning microSD card with Raspbian loaded on it!

## XBee Readiness

Next, we'll configure the XBee devices in order to ensure they can work with the xbee Python library used
in the tutorial. In this tutorial, we used an Arduino Duemilanove with an attached XBee
[Shield](https://shieldlist.org/libelium/xbee) (details can be found [here](https://www.arduino.cc/en/Main/ArduinoXbeeShield)).
In order for the XBee to be programmable through the Arduino-attached XBee shield, the ATMega
chip on the Arduino board must be either removed or shorted. The easiest way to accomplish this
is to connect the RESET and GND connections on the Arduino with a single wire, thereby bypassing
the ATMega chip.

Once the ATMega chip has been bypassed and the XBee Shield (and associated XBee device) have been
connected, plug a USB cable into the Arduino board and connect it to a computer running the XCTU
software. Once connected, in XCTU, select "Add a radio module". In the dialog that appears, you should see
your XBee show up (on a MacBook Pro, this shows as something similar to "usbserial-A400fGEG"). Select
the USB device, leaving all other settings as default, and select "Finish".

When the configuration window shows (gear icon selected in the upper right corner, and XBee device
selected in left device selector), click the "Default" icon (looks like a factory for factory settings)
and proceed to the next step below...

**THIS NEXT STEP IS IMPORTANT**

The XBee must be configured for API mode. Under the "Serial Interfacing" section in the configuration
settings, update the "AP API Enable" parameter to be "API enabled [1]". This puts the XBee in API,
no-escape mode, which is what the Xbee Python library expects (and is required if you end up having more
than 2 XBee devices at a time).

If you wish to configure a specific channel and PAN ID for the network of devices you're building, feel
free to do so (just ensure both XBee modules have the same PAN ID and corresponding Channel). This will help
if you have or plan to have separate XBee networks and do not want cross-talk/communication.

Once completed, click the "Write" icon at the top of the configuration page to write the changes to
the XBee module.

Repeat the same exact steps as above for your second XBee module. Ideally, you would have the first *and*
second module plugged in at the same time and configured so you can use the "Console" feature of XCTU
to verify that messages are sent/received by each XBee, indicating they have been correctly configured.

## Circuit Wiring

You can now move on to wiring the XBee modules according to the circuit diagrams below:

### Transmitter

[![Transmitter Circuit][1]][1]

For the control/button (transmitter) circuit (Raspberry PI B+ - First Instance):

**Buttons**:

- UP button PIN1 to RasPi1 GPIO19 (PIN35)
- DOWN button PIN1 to RasPi1 GPIO13 (PIN33)
- LEFT button PIN1 to RasPi1 GPIO6 (PIN31)
- RIGHT button PIN1 to RasPi1 GPIO5 (PIN29)
- UP button PIN2 to RasPi1 GND
- DOWN button PIN2 to RasPi1 GND
- LEFT button PIN2 to RasPi1 GND
- RIGHT button PIN2 to RasPi1 GND

**XBee**:

- XBee VCC to RasPi1 +3.3V
- XBee GND to RasPi1 GND
- XBee DOUT to RasPi1 UART0_RXD (PIN10)
- XBee DIN to RasPi1 UART0_TXD (PIN8)

### Receiver

[![Receiver Circuit][2]][2]

For the LED (receiver) circuit (Raspberry PI 3 B+ - Second Instance):

**LEDs**:

- UP LED NEG to 270 Ohm Resistor to RasPi2 GND
- UP LED POS to RasPi2 GPIO21 (PIN40)
- DOWN LED NEG to 270 Ohm Resistor to RasPi2 GND
- DOWN LED POS RasPi2 GPIO20 (PIN38)
- LEFT LED NEG to 270 Ohm Resistor to RasPi2 GND
- LEFT LED POS RasPi2 GPIO16 (PIN36)
- RIGHT LED NEG to 270 Ohm Resistor to RasPi2 GND
- RIGHT LED POS RasPi2 GPIO12 (PIN32)

**XBee**:

- XBee VCC to RasPi2 +3.3V
- XBee GND to RasPi2 GND
- XBee DOUT to RasPi2 UART0_RXD (PIN10)
- XBee DIN to RasPi2 UART0_TXD (PIN8)

## Raspberry Pi Device Readiness

In order for the Raspberry Pi to be able to use the UART ports (GPIO14/PIN8 and GPIO15/PIN10), they
first need to be removed from serial console use (default for the Raspbery Pi). In order to do this,
launch the Raspberry Pi configuration utility:

{% highlight bash %}
$ sudo raspi-config
{% endhighlight %}

Once the configuration utility appears, navigate to "5 Interfacing Options" -> "P6 Serial", and
select "No" to disable all options. Quit the configuration utility, but do *not* elect to reboot (yet).

Once the Serial console has been disabled, ensure the UART is enabled. Open the `config.txt` file and
search for `enable_uart`, and ensure it is set to "1":

{% highlight bash %}
$ sudo vim /boot/config.txt
# search for and ensure the following line exists:
#   enable_uart=1
{% endhighlight %}

Once the above have completed, trigger a reboot of the Raspberry Pi for the new settings to
take effect:

{% highlight bash %}
$ sudo reboot
{% endhighlight %}

Perform the above steps in their entirety for each Raspberry Pi instance.

## Python Environment, Library Dependencies, Code, and Execution

Now that we have the hardware setup and configuration out of the way, we can start to do some programming.
This tutorial *requires* Python 3 to function given the libraries used within only being compatible with
Python 3. We will walk through setting up a Python Virtual Environment, installing the respective libraries
required, and coding the transmitter and receiver functionality.

Note that the steps involved can be executed on one of the Raspberry Pi instances, saved/stored in GitHub
(or some other source repository), and then pulled down for execution on the secondary Raspberry Pi instance.
It does not matter which Raspberry Pi instance you use for developing the source code, so long as they both
have access to the source code repository being used to store the code.

If you wish to get a starter copy of the code, you can clone the [following repository](https://github.com/jekhokie/raspi-projects)
and take the scripts (or entire project) from the `11-xbee-send-receive` folder.

### Virtual Environment Setup

In order to help manage the libraries/dependencies adequately for the project, we will use the virtual
environment functionality in Python. To create a project folder (where all of your code will be stored)
and create a virtual environment, perform the following on one of the Raspberry Pi instances:

{% highlight bash %}
# install virtualenv for Python 3
$ sudo pip3 install virtualenv

# create a project directory and virtual environment
$ mkdir xbee-tutorial
$ cd xbee-tutorial/
$ virtualenv .env

# activate your terminal/session for the virtual env
# this will set the respective python paths required
# for the libraries to be loaded accurately from the
# '.env' directory
$ .env/bin/activate

# verify you are using Python 3 in your virtual env
# this is required for the code we will produce
$ python --version
# should output something similar to:
#   Python 3.5.3
{% endhighlight %}

### Dependencies

We will use a `requirements.txt` file to track our libraries/dependencies. Create this file named
`requirements.txt` and ensure it has the following contents:

```bash
xbee
RPi.GPIO
```

### Code

Next, we'll develop the transmitter code (the controller with the buttons). Create a file named
`transmitter.py` and ensure it has the following contents:

```python
#!/usr/bin/env python3
#
# Acts as a transmitter of data to a receiving XBee module. Sends commands
# based on push-buttons connected to the device.
#
# **NOTE: REQUIRES PYTHON 3**

# import libraries
import RPi.GPIO as GPIO
import serial
import time
from xbee import XBee

# assign the button pins and XBee device settings
BUTTON_UP_PIN = 19
BUTTON_DOWN_PIN = 13
BUTTON_LEFT_PIN = 6
BUTTON_RIGHT_PIN = 5
SERIAL_PORT = "/dev/ttyS0"
BAUD_RATE = 9600

# set the pin mode
GPIO.setmode(GPIO.BCM)
GPIO.setwarnings(False)

# initialize GPIO inputs for buttons
for b in [BUTTON_UP_PIN, BUTTON_DOWN_PIN, BUTTON_LEFT_PIN, BUTTON_RIGHT_PIN]:
    GPIO.setup(b, GPIO.IN, pull_up_down=GPIO.PUD_UP)

# configure the xbee
ser = serial.Serial(SERIAL_PORT, baudrate=BAUD_RATE)
xbee = XBee(ser, escaped=False)

# handler for sending data to a receiving XBee device
def send_data(data):
    xbee.send("tx", dest_addr=b'\x00\x00', data=bytes("{}".format(data), 'utf-8'))

# initialize previous states
last_up_state = False
last_down_state = False
last_left_state = False
last_right_state = False

# send data to ensure all LEDs are off starting out
for i in ["0UP", "0DOWN", "0LEFT", "0RIGHT"]:
    send_data(i)

# main loop/functionality
while True:
    try:
        # obtain current state of each button
        up_state = GPIO.input(BUTTON_UP_PIN)
        down_state = GPIO.input(BUTTON_DOWN_PIN)
        left_state = GPIO.input(BUTTON_LEFT_PIN)
        right_state = GPIO.input(BUTTON_RIGHT_PIN)

        # check if button changed for "up"
        if up_state != last_up_state:
            last_up_state = up_state
            if up_state == False:
                print("Pressed up")
                send_data("1UP")
            else:
                print("Released up")
                send_data("0UP")

        # check if button changed for "down"
        if down_state != last_down_state:
            last_down_state = down_state
            if down_state == False:
                print("Pressed down")
                send_data("1DOWN")
            else:
                print("Released down")
                send_data("0DOWN")

        # check if button changed for "left"
        if left_state != last_left_state:
            last_left_state = left_state
            if left_state == False:
                print("Pressed left")
                send_data("1LEFT")
            else:
                print("Released left")
                send_data("0LEFT")

        # check if button changed for "right"
        if right_state != last_right_state:
            last_right_state = right_state
            if right_state == False:
                print("Pressed right")
                send_data("1RIGHT")
            else:
                print("Released right")
                send_data("0RIGHT")

        time.sleep(0.2)
    except KeyboardInterrupt:
        break

# clean up
GPIO.cleanup()
xbee.halt()
ser.close()
```

The code above is self-documenting, but the premise is to set up the Raspberry Pi GPIO pins, initialize
them, and then read each of the pins to determine if there was a state change (and transmit the state change
to the receiver). The transmission is using the following encoding scheme:

`<STATE><DIRECTION>`

Where:

- **\<STATE\>**: Ether 1 or 0, indicating button is pressed or not pressed, respectively
- **\<DIRECTION\>**: One of "UP", "DOWN", "LEFT", "RIGHT" indicating which button has the state change

The code could be optimized further but for readability and some debugging purposes, we will leave it as
noted above.

Next, we'll create the receiver code. Create a file named `receiver.py` with the following contents:

```python
#!/usr/bin/env python3
#
# Acts as a receiver of data from a sending XBee module, and controls
# LEDs (up, down, left, right) associated with the codes sent by the trasmitter.
#
# **NOTE: REQUIRES PYTHON 3**

# import libraries
import serial, time
import RPi.GPIO as GPIO
from xbee import XBee

# assign the LED pins and XBee device settings
LED_UP_PIN = 21
LED_DOWN_PIN = 20
LED_LEFT_PIN = 16
LED_RIGHT_PIN = 12
SERIAL_PORT = "/dev/ttyS0"
BAUD_RATE = 9600

# set the pin mode
GPIO.setmode(GPIO.BCM)
GPIO.setwarnings(False)

# initialize GPIO outputs for LEDs
for l in [LED_UP_PIN, LED_DOWN_PIN, LED_LEFT_PIN, LED_RIGHT_PIN]:
    GPIO.setup(l, GPIO.OUT)

# handler for whenever data is received from transmitters - operates asynchronously
def receive_data(data):
    print("Received data: {}".format(data))
    rx = data['rf_data'].decode('utf-8')
    state, led = rx[:1], rx[1:]

    # parse the received contents and activate the respective LED if the data
    # received is actionable
    if state in ("0", "1"):
        if led in ("UP", "DOWN", "LEFT", "RIGHT"):
            if led == "UP":
                GPIO.output(LED_UP_PIN, int(state))
            elif led == "DOWN":
                GPIO.output(LED_DOWN_PIN, int(state))
            elif led == "LEFT":
                GPIO.output(LED_LEFT_PIN, int(state))
            elif led == "RIGHT":
                GPIO.output(LED_RIGHT_PIN, int(state))
        else:
            print("ERROR: Received invalid LED '{}' - ignoring transmission".format(state))
    else:
        print("ERROR: Received invalid STATE '{}' - ignoring transmission".format(state))

    print("Packet: {}".format(data))
    print("Data: {}".format(data['rf_data']))

# configure the xbee and enable asynchronous mode
ser = serial.Serial(SERIAL_PORT, baudrate=BAUD_RATE)
xbee = XBee(ser, callback=receive_data, escaped=False)

# main loop/functionality
while True:
    try:
        # operate in async mode where all messages will go to handler
        time.sleep(0.001)
    except KeyboardInterrupt:
        break

# clean up
GPIO.cleanup()
xbee.halt()
ser.close()
```

Again, the code is pretty self-documenting (not best practice per Python standards, but good enough for
this tutorial), but the premise is to again initialize the pins and loop continuously, attempting to
receive data from the transmitter, decode the message (per the format listed earlier), and produce the
appropriate state on the up/down/left/right LED GPIO pins. There is again some code optimization and
meta-programming that could be used to clean this up and shorten it, but we will leave it as-is in the
interest of readability.

These 3 files are essentially all you will need to execute the transmitter/receiver functionality. Commit
the code to your source repository, and then clone the repository/pull the code down onto your secondary
Raspberry Pi instance (the one that you haven't yet put any code onto).

### Execution

Now that we have our code base, let's kick off the demo. Assuming you have the code locally on each of the
2 Raspberry Pi instances (transmitter and receiver), you've wired the circuits correctly, and you've configured
both Raspberry Pi UART/Serial settings according to the above, you should be able to kick off the test.

First, let's launch the **receiver** functionality. Assuming you are on the receiver Raspberry Pi and in the
project directory already, let's go ahead and create the Python Virtual Environment (skip this step if this is
the Raspberry Pi you've already created the Virtual Environment on):

{% highlight bash %}
# install virtualenv for Python 3
$ sudo pip3 install virtualenv

# change to your project directory and create the
# virtual env
$ cd xbee-tutorial/
$ virtualenv .env

# activate your terminal/session for the virtual env
# this will set the respective python paths required
# for the libraries to be loaded accurately from the
# '.env' directory
$ .env/bin/activate

# verify you are using Python 3 in your virtual env
# this is required for the code we will produce
$ python --version
# should output something similar to:
#   Python 3.5.3
{% endhighlight %}

Next, we will install the dependencies (if this is the Raspberry Pi you've already done this on, it's safe to
run this command again):

{% highlight bash %}
$ pip install -r requirements.txt
{% endhighlight %}

We'll now launch the receiver code - you can launch it as a background job, cron, init script, or whatever
other method you'd like down the road, but for simplicity and debugging we will simply launch it in the
foreground for now and keep it running for debugging purposes:

{% highlight bash %}
$ python receiver.py
{% endhighlight %}

At this point, you should see nothing happening on the screen (until we get the transmitter going). Repeat the
above steps in their entirety for the transmitter Raspberry Pi instance, except when executing the launch
command, use `python transmitter.py` instead of the 'receiver' script.

### Testing

Both the transmitter and the receiver are now functioning. You should be able to press any of the directional
buttons and see the corresponding/respective LED light up on the receiver end, indicating your circuits are
wired correctly, Raspberry Pi's are configured appropriately, and the XBees are communicating!

## Troubleshooting

It took many iterations to finally get this working, and some easy mistakes were made in the sea/mess of
wires given a shared breadboard was being used for both Raspberry Pi and XBee instances as well as all
of the electronic components. Here are some quick-hit bullets to check if you find the circuit is not working
as you expect:

1. Ensure the Raspberry Pi UART TX (Transmit) is connected to the XBee RX (Receive), and the corresponding
Raspberry Pi UART RX (Receive) is connected to the XBee TX (Transmit). Messing this up will result in no
messages being passed by the Raspberry Pi/XBee pair.
2. Double-check your UART settings in the OS. It's easy to forget to edit the files to enable UART mode.
3. Ensure your `/dev/ttyS0` device is present/available. Legacy Raspberry Pi instances would have the
`/dev/ttyAMA0` device listed as the UART port, but on newer Raspberry Pi Operating Systems, this should be
`/dev/ttyS0` (as listed in the code). If the device is not present, it's likely that you missed editing the
`/boot/config.txt` file (or forgot to restart your Raspberry Pi instance so the device settings would take
effect).
4. All grounds connected! Again, in the sea of wires (and split breadboard VCC/GND rails) it's easy to miss
a ground connection. Ensure the Raspberry Pi, XBee, and circuit electronics all share a common ground.

If all else fails, attempt to go back to basics. Start by interfacing both XBee instances in XCTU and test
that messages can be sent between them through the XCTU application (in both directions). Then take one of
the XBee instances and connect it to one of the circuits (transmitter or receiver) and attempt to run the
`transmitter.py` or `receiver.py` scripts and send/receive messages between the circuit and the XBee connected
to the XCTU application to ensure your packets are sent appropriately. Then move the final XBee from XCTU
back into the circuit. If things are still not working, double/triple check your wiring to ensure it matches
the diagrams in this tutorial.

## Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [XBee for Arduino and Raspberry Pi](https://www.cooking-hacks.com/documentation/tutorials/xbee-arduino-raspberry-pi-tutorial)
* [Raspberry Pi](https://www.raspberrypi.org/)
* [XBee User Guide](https://www.digi.com/resources/documentation/digidocs/pdfs/90000982.pdf)
* [Etcher](https://www.balena.io/etcher/)
* [Connecting XBee to Raspberry Pi](https://dzone.com/articles/connecting-xbee-raspberry-pi)
* [Analyzing sensor readings with an XBee wireless connection](http://www.raspberry-pi-geek.com/Archive/2015/12/Analyzing-sensor-readings-with-an-XBee-wireless-connection)
* [Building a wireless temperature sensor with a Raspberry Pi and Xbee wireless modules](http://www.brettdangerfield.com/post/raspberrypi_tempature_monitor_project/)
* [XBee Python Module](https://pypi.org/project/XBee/)
* [Getting Started with XBee Python Library](https://xbplib.readthedocs.io/en/latest/getting_started_with_xbee_python_library.html)
* [Raspbian](https://www.raspberrypi.org/downloads/raspbian/)

[1]: /assets/images/2018-12-30-raspberry-pi-xbee-rxtx-transmitter.png
[2]: /assets/images/2018-12-30-raspberry-pi-xbee-rxtx-receiver.png
