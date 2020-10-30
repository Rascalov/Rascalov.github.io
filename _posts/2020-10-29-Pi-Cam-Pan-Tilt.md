---
layout: post
title:  "Raspberry Pi Pan-Tilt surveillance Camera"
date:   2020-10-29 09:59:21 +0200
categories: moodle java javafx scraper
toc: true
tags:
 - Raspberry-Pi
 - Camera
 - Streaming
 - DIY
 - Pan-Tilt
---

# Pi cam: Hindsight and a massive upgrade
Last time I used the pi cam, I set it up in a cheap plastic/cardboard box and put it outside. 
This, sadly, was not up to the task given the weather we have down here.
The angle was also not satisfactory at all times, given the sun moving higher and lower 
depending on the day. 

The setup also lacked something I really wanted: pan tilt functionality. Moving the camera around remotely turns me on. I decided to look into the options.
Also its usage, I wanted 24/7 surveillance of my front door and derivatives.


I will sum up my needs:
* 24/7 live stream, nothing recorded (as per the law at my district of residence)
* Pan-tilt functionality, camera can move up, down, left and right
* Must be viewable (pc, mobile, tablet) by the entire family, not just 1 client, so no netcat.

24/7 stream should be trivial, I have a small heatsink on the cpu and ethernet chip. Coming 
weather will be cold. All I need is housing for the pi that is weatherproof.

To accomplish this, I bought a squirrel feeder from amazon for 15 €
 

So I will 
 
 
$sudo uv4l -nopreview --auto-video_nr --driver raspicam --encoding mjpeg --width 640 --height 480 --framerate 20 --server-option '--port=9090' --server-option '--max-queued-connections=30' --server-option '--max-streams=25' --server-option '--max-threads=29'


Turns out there are a lot of options. Most refer to Servo Motors, simple motors that can turn 180 degrees. I can set
two servos, one takes the left and right, the other takes up and down.

I bought some servos, along with a kit to help me set up a HAT I can use to put the pan tilt setup on the raspberry pi.

