---
layout: post
title:  "Raspberry Pi Camera"
date:   2020-10-20 09:59:21 +0200
categories: moodle java javafx scraper
toc: true
tags:
 - Raspberry-Pi
 - Camera
 - Streaming
 - DIY
---

# Raspberry Pi camera setup
I bought a camera add on and made it connect with netcat. So I have a live stream with ~3 ms latency. 

## use in ssh remote access to raspberry pi 
```raspivid -t 0 -w 1280 -h 720 -o - | nc 192.168.0.100 5000```

## use in target linux system
```netcat -l -p 5000 | mplayer -fps 60 -cache 2048 -```

results in barely any latency, now I have to get it outside to watch the front door. 

## Front door setup
I put my pi in a plastic case and put that in a medium-sized box I got from an action figure sale. 
The figure was a chinese knockoff, but the box was decent. 

Here's the box:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/index.jpeg" width="500" height="350">


With an opening:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/flatbox.jpeg" width="500" height="350">


