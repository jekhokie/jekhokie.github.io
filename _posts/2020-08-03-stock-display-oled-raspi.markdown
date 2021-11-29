---
layout: post
title:  "Displaying Stock Price on OLED Display using RasPi"
date:   2020-08-03 20:18:00 -0400
categories: raspberry-pi oled python linux
logo: stock-watcher-oled.jpg
---

Sometimes, it's just helpful to have something front and center vs. having to navigate to a web page to find it. This tutorial uses a
Raspberry Pi Zero and an OLED display to show a near real-time stock price and gain/loss information that can be used anywhere that the
Raspberry Pi can connect to a network with internet connectivity.

## Limitations

Obtaining pre and post-market data can be a bit more complicated than simply obtaining during-hours data. The Yahoo finances library
used here does not yet support off-hours quotes, so the data returned off-hours is the last close price of the stock. This is going to be
left as an enhancement to the reader's efforts.

## Parts Needed

This tutorial utilizes the following parts and software:

- 1x Raspberry Pi Zero WH (almost any Raspberry Pi version should work)

**WARNING**: It is recommended that you utilize an external +3.3V power supply and *not* the VCC from the Raspberry Pi instance to
power the OLED display. The Raspberry Pi can consume quite a bit of power and this may interrupt proper operation of the OLED display
under certain circumstances.

## Prerequisites

It is assumed the reader understands how to install/configure an adequate operating system for the Raspberry Pi such as Raspberry Pi OS,
etc. This tutorial assumes the Raspberry Pi OS is used - while other operating systems may in fact work with the code provided (Python),
if things don't work as expected, attempt to install/configure Raspberry Pi OS with the provided code to verify it's not a difference in
the operating system that is causing the issue..

## Wiring/Connectivity

First, the wiring. Connect the OLED display to the Raspberry Pi according to the following diagram - in summary, 3.3V to RasPi VCC,
GND to RasPi GND, SDA to RasPi SDA, and SCL to RasPi SDL:

[![Wiring Circuit][1]][1]

## Enabling I2C

This code/example uses I2C to communicate with the OLED display. To enable I2C, SSH to the Raspberry Pi device and run `sudo raspi-config`,
then navigate to `Interfacing Options`. Select `I2C` and select `<Yes>` when prompted to enable I2C. This may require a reboot to enable - go
ahead and reboot the Raspberry Pi device.

## Programming

Now that I2C is enabled, SSH to the Raspberry Pi again and create a directory for the source code and install some required depencies/create
a Python virtual environment:

```bash
# install os dependencies
$ sudo apt-get -y install python3-pandas

# create a project directory
$ mkdir stock-watcher
$ cd stock-watcher/

# create a python3 virtual environment and activate it
$ pip3 install virtualenv
$ python3 -m virtualenv .env
$ . .env/bin/activate

# install python dependencies
$ pip install adafruit-blinka \
              adafruit-circuitpython-ssd1306 \
              html5lib \
              Pillow \
              requests \
              requests_html \
              yahoo_fin
$ pip --no-cache-dir install pandas
```

Now that the dependencies are installed, go ahead and create a file named `display_stock.py` with the following contents:

```python
#!/usr/bin/env python3

import adafruit_ssd1306
import board
import busio
import digitalio
import socket
import subprocess
import time
from datetime import datetime
from PIL import Image, ImageDraw, ImageFont
from yahoo_fin import stock_info

# which stock to track
TICKER_SYMBOL = 'AAPL'

# configs
OLED_ADDR = 0x3C
OLED_WIDTH = 128
OLED_HEIGHT = 64

# initialize i2c and bus
i2c = busio.I2C(board.SCL, board.SDA)
oled = adafruit_ssd1306.SSD1306_I2C(OLED_WIDTH, OLED_HEIGHT, i2c, addr=OLED_ADDR)

# clear display
oled.fill(0)
oled.show()

# create blank image for drawing, and initialiize the draw/fonts
image = Image.new("1", (OLED_WIDTH, OLED_HEIGHT))
draw = ImageDraw.Draw(image)
ticker_font = ImageFont.truetype("/usr/share/fonts/truetype/freefont/FreeSans.ttf", 16)
price_font = ImageFont.truetype("/usr/share/fonts/truetype/freefont/FreeSans.ttf", 26)
diff_font = ImageFont.truetype("/usr/share/fonts/truetype/freefont/FreeSans.ttf", 12)

try:
    while True:
        print("Querying stock...")

        # get the date/time for update
        update_datetime = datetime.now()
        update_time = update_datetime.strftime('%I:%M:%S')

        # get stock information
        try:
            quote = stock_info.get_quote_table(TICKER_SYMBOL)
            price = round(quote['Quote Price'], 2)
            prev_close = round(quote['Previous Close'], 2)
            delta = round(price - prev_close, 2)
            symbol = '\u25b2' if delta >= 0 else '\u25bc'   # display up/down arrow based on movement

            # convert to strings to display/handle display
            price_str = str(price)
            delta_str = str(delta)

            # calculate positioning
            (price_font_width, price_font_height) = price_font.getsize(price_str)
            (diff_font_width, diff_font_height) = diff_font.getsize(delta_str)

            # clear and draw stock info to screen
            draw.rectangle((0, 0, OLED_WIDTH, OLED_HEIGHT), outline=0, fill=0)
            draw.text((0, 0), TICKER_SYMBOL, font=ticker_font, fill=255)
            draw.text((oled.width - diff_font.getsize(update_time)[0], 0), update_time, font=diff_font, fill=255)
            draw.text((oled.width // 2 - price_font_width // 2, oled.height // 2 - price_font_height // 2), "${}".format(price_str), font=price_font, fill=255)
            draw.text((oled.width // 2 - diff_font_width // 2, OLED_HEIGHT - diff_font_height), "{} ${}".format(symbol, delta_str), font=diff_font, fill=255)
            oled.image(image)
            oled.show()

            print("Done!")
        except socket.gaierror as e:
            print("Failure to get stock quote - DNS error: {}".format(str(e)))
        except Exception as e:
            print("Failure to get/process stock quote - unknown error: {}".format(str(e)))

        # sleep between queries
        time.sleep(5)
except KeyboardInterrupt:
    print("SIGINT detected - releasing screen")
    oled.fill(0)
    oled.show()
    exit(0)
```

Once the above is created, update the `TICKER_SYMBOL` value and run the script. If all goes well, you should see the stock symbol,
current price, and change since last close price display on the OLED!

## Enhancing

Now that you have your script created and working, there are many things you could do to enhance it (add buttons for different metrics,
add lights when you hit an all-time high, etc.). In addition, MUCH more error checking should be added to the script above.

One useful thing is to enable the Raspberry Pi to launch the stock symbol display on boot so you don't need to run the script by hand each time the Raspberry Pi
is rebooted. To do this, edit the file `/etc/rc.local` and add the following lines prior to the `exit 0` directive:

```bash
$ sudo vim /etc/rc.local
# add the following before `exit 0`:
#   ...
#   . /home/pi/stock-watcher/.env/bin/activate
#   python3 /home/pi/stock-watcher/display_stock.py
#   exit 0
```

Now reboot the Pi device. If all goes well, the OLED should automatically be populated with the stock information once the Pi device boots,
gets an IP address, and is able to connect to the internet. If something appears to have gone wrong, investigate the logs in `/var/log` for
clues.

## Troubleshooting

There is a possibility when running the `display_stock.py` script you will get an error related to the `numpy` package (especially if you're running on a Pi
Zero or similar) that looks like the following:

```bash
Traceback (most recent call last):
  File "display_stock.py", line 10, in <module>
    from yahoo_fin import stock_info
  File "/home/pi/stock-watcher/.env/lib/python3.7/site-packages/yahoo_fin/__init__.py", line 3, in <module>
    import pandas as pd
  File "/home/pi/stock-watcher/.env/lib/python3.7/site-packages/pandas/__init__.py", line 17, in <module>
    "Unable to import required dependencies:\n" + "\n".join(missing_dependencies)
ImportError: Unable to import required dependencies:
numpy:

IMPORTANT: PLEASE READ THIS FOR ADVICE ON HOW TO SOLVE THIS ISSUE!

Importing the numpy C-extensions failed. This error can happen for
many reasons, often due to issues with your setup or how NumPy was
installed.

We have compiled some common reasons and troubleshooting tips at:

    https://numpy.org/devdocs/user/troubleshooting-importerror.html

Please note and check the following:

  * The Python version is: Python3.7 from "/home/pi/stock-watcher/.env/bin/python"
  * The NumPy version is: "1.19.4"

and make sure that they are the versions you expect.
Please carefully study the documentation linked above for further help.

Original error was: libf77blas.so.3: cannot open shared object file: No such file or directory
```

A potential fix is documented in the link detailed in the error - what worked well on the Pi Zero that was being used when this was written was to
perform the following, which resulted in being able to then successfully launch the `display_stock.py` script:

```bash
$ sudo apt-get install libatlas-base-dev
```

### Credit

The above tutorial was pieced together with some information from the following sites/resources, among others that were likely missed in this list:

- [Python Usage](https://learn.adafruit.com/monochrome-oled-breakouts/python-usage-2)
- [Command to Display Memory Usage, Disk Usage, and CPU Load](https://unix.stackexchange.com/questions/119126/command-to-display-memory-usage-disk-usage-and-cpu-load)

[1]: /assets/images/2020-08-03-stock-display-oled-raspi-circuit.png
