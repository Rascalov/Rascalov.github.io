# Rascalov's Progress Manifesto
Ik ben Tim, en het maken van een site duurt te lang. Dus ga ik mijn vooruitgang op github pages documenteren.

Ik stel een paar kopjes op, deze worden, in de loop van tijd, gevuld met wat er is geleerd.

## Routing (Netwerken 2)
Herkansing komt eraan, heb weinig geleerd.
### Static routing
Van wat ik lees, routers kunnen niet zomaar van netwerk naar netwerk hoppen zonder een routing table waarin het destination adres staat.
Dus per netwerk moet er gezegd worden naar welke directly connected router het naartoe moet. 

Dynamic routing (rip2) maakt dit makkelijker door direct verbonden routers elkaars routing tables te sturen.



## Moodle Automatic File Retriever (MAFR)
2020-06-30

I realized that no one will ever use my creation. No matter what source code I hand over and what I might say, people are not comfortable
inserting their credentials. 

I keep reminding myself of this every time I look at MAFR. It's the only thing I made that I felt kinda good about. Doing a GTAV mod was cool,
but nothing special. MAFR could help a lot of future students of Inholland University of Applied Sciences. If only I would polish it.

So I will, but there are challenges.

### What needs to be done?
An answer I do not have, but I have theories:

* Remove the login requirement
* Create a graphical interface
* Optimize the file acquisition proces 

Removing the login is what I believe will help directly. 

This does beg the question: "How the fuck would anything work then?"

I cannot just give session cookies to clients from my own login. Any person could just put those credentials into their browser
and fuck my account up. 

From what I read, this is a situation where the use of a **server-client infrastructure** is warranted.  

#### What does MAFR do again?
In order to figure out what the **Server's** and **Client's** responsibilities are, I need to recap to myself the idea of MAFR.

First the login, then check if there's access to the course, if not, enroll. Then determine the structure of the course. 
Then check all files per section and index them. Finally, per file, we attempt to download the content.

Updating follows more or less the same steps, except it checks the last-modified date before downloading. 
  

#### Server
The server has the credentials. Users can set up their own or use the one I set up.
Server is responsible for:
* Login
* Determining and returning the folder structure
* Return content of a section
* Handle and download requested links

#### Client
The Client's controlled by the end-user. It'll communicate with the server through api calls. 
Client must enter the server url and the password of the server. Which the server host should give. 
Apart from that, the client's only purpose is to send requests handle the responses given by the server. 

2020-07-01

The client's UI is still up for debate, even though it is probably going to be very straightforward and (decently) clean.

A few logos, some input and you got half the appeal and functionality. 

The folder structure determination that I implemented on MAFR 1.5 is an unstable and inefficient way to determine.
Improving it means I have to concentrate on the html. More than I ever did. My brain wasn't made for looking at html with 
obfuscated classes and id's.

I give the user 2 options:
* Literal: Folder structure exactly as it's shown on Moodle
* Simple: Section folders only have 1 subfolder level.

Main reason for these options is because the ones that create the courses have a hard time structuring it properly, 
so the user has to decide which option will work for the course.  

#### The structure
This image explains most of it:
<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/moodleStructure.jpeg" width="500" height="350">


 
 

## ITSM theorie
Belangrijk zijn de anki kaarten die Bastiaan heeft gemaakt. Leer die en pak wat extra info van de powerpoints
## NoSQL theorie
Anki kaarten en de vragen van de leerlingen leren. 
## Java Fundamentals 
## Deskresearch
## Linux (bash, shell scripts, distro's, etc.)
## VIM 
## Russisch
## Raspberry Pi camera setup
I bought a camera add on and made it connect with netcat. So I have a live stream with ~3 ms latency. 

### use in ssh remote access to raspberry pi 
```raspivid -t 0 -w 1280 -h 720 -o - | nc 192.168.0.100 5000```

### use in target linux system
```netcat -l -p 5000 | mplayer -fps 60 -cache 2048 -```

results in barely any latency, now I have to get it outside to watch the front door. 

### Front door setup
I put my pi in a plastic case and put that in a medium-sized box I got from an action figure sale. 
The figure was a chinese knockoff, but the box was decent. 

Here's the box:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/index.jpeg" width="500" height="350">


With an opening:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/flatbox.jpeg" width="500" height="350">


   






