---
layout: post
title:  "SDR Streaming using RTMP"
date:   2021-03-03 19:34:00 -0400
categories: sdr rtmp
logo: sdr-rtmp.jpg
---

Software Defined Radio (SDR) devices are incredibly flexible devices allowing all kinds of interesting hobbies such as satellite tracking,
radio listening, etc. This post details how to publish the captured signals from a SDR device to a RTMP endpoint that allows near real-time
streaming (listening) to the device. The approach is flexible such that it allows multiple/many listeners to attach to the stream at one
time.

## Architecture

When complete, this tutorial will result in the SDR device piping the received radio data to an Nginx RTMP endpoint hosted on the Pi. This
Nginx RTMP streaming server will be the point for multiple consumers to consume the stream being published. The laptop on the same network will
then use a command-line binary `rtmpdump` to connect to the same Nginx instance hosted on the Raspberry Pi and play this streaming audio through
the `vlc` player.

It is assumed that the `rtmpdump` and `vlc` software are installed on the laptop (external device).

## Parts Needed

This tutorial assumes that you have a SDR device attached to a Debian-based operating system either on a typical computer or microcomputing
device such as a Raspberry Pi. For the sake of simplicity, the tutorial will walk thorugh setting this up with the SDR attached to an antenna
and plugged into a Raspberry Pi 4 device, connected to a wireless network from which a laptop (MacBook) can also reach the Pi device.

To complete this tutorial, you will need:

1. Software Defined Radio (SDR) dongle, such as the RTL-SDR V3 found [here](https://www.rtl-sdr.com/buy-rtl-sdr-dvb-t-dongles/)
2. Antenna that can connect to the SDR dongle purchased.
3. Device to host the SDR device (Raspberry Pi 4 or similar with Debian-based distribution installed)
4. Computer connected to the same network as the SDR host device.

This tutorial also assumes you have successfully connected and enabled your SDR dongle to your host device. There are many tutorials
on how to approach this, so this step will be left out of this tutorial/assumed completed.

## Software Prerequisites

There are a few prerequisites required on the Raspberry Pi device to get things set up. Again, it is assumed the SDR device and RTL tools
are already installed and functioning. The prerequisites required can be installed via the following:

```bash
$ sudo apt-get -y install nginx \
                          libnginx-mod-rtmp \
                          ffmpeg
```

Once the above complete, edit the file `/etc/nginx/nginx.conf` and add the following stanza at the root level of the config file:

```
rtmp {
  server {
    listen 1935;
    chunk_size 4096;

    application live {
      live on;
      record off;
    }
  }
}
```

The above configuration enables RTMP listening on port 1935. The incoming RTMP messages will be live streamed without recording of the
data.

Next, restart Nginx for the settings to take effect: `sudo systemctl restart nginx.service`

## Start Stream

Now that the Nginx RTMP endpoint is available for streams, start the stream by running `rtl_fm` tuned to an FM frequency that is available
in your local region. Note that we are only using wideband-FM (typical FM radio) as an example - any frequency range can work for this
setup:

```bash
$ rtl_fm -f 107.9e6 -M wbfm -s 200k -A std -l 0 -r 48k - | ffmpeg -f s16le -ac 1 -i pipe:0 -ab 128k -f flv rtmp://localhost:1935/live/test
```

The above will take the FM station 107.9 FM and pipe it to `ffmpeg`, which will stream to the RTMP endpoint. Your live stream is now
available through the Nginx RTMP endpoint.

## Consume Stream

Now that the stream is available for consumption, switch back to the laptop/external device and run the following command, replacing
`<YOUR_PI_IP>` with the IP or resolvable hostname of your Raspberry Pi device:

```bash
$ rtmpdump -v -r "rtmp://<YOUR_PI_IP>:1935/live/test" -o - | vlc -
```

The above will consume the RTMP stream from the Raspberry Pi Nginx endpoint and pipe it to the local VLC player. If all goes well, you should
be hearing the local radio station! (worst case, you should hear static if your antenna is not sufficient or the station is not available)

## Next Steps

Now that you have an RTMP streaming server in place and a way to send audio from local airwaves to it, the possibilities are endless. This
configuration allows for multiple consumers at the same time, meaning you can have multiple endpoints all accessing the radio streams. The
only limitation is the available processing power for the Raspberry Pi device to take on however many concurrent connections you want to
throw at it!
