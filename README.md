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

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/moodleStructure.png" width="600" height="350">

Then, once you select a course subject, it opens the submenus:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/moodleStructure2.png" width="600" height="350">

The script calls distinguish the subfolders by giving them section-Id's. The attribute calls them like so: ``data-section=1``    

This is a regular situation, you might've already guessed based on the ``list-group-submenu-level-1`` that subfolders can have their own subfolders. Level-2, 3 etc. So you have to take that into account.

Below is an example:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/moodleStructure3.png" width="600" height="350">

Notice how level 2 is enveloped in a ``<div>`` instead of the usual ``<a>``.
That means his section-id and potential submenus are present there.

((Bonus: The picture above is what I mean when I say the course creators fuck up the 
subfolders. Papers/journal articles has no place underneath the slides folder.
It should've been a level-1 folder.)) 

Given these pictures, this means that, per section, we must check for multiple levels.
Once the levels are counted, you can loop the folder creation until the levels run out. 

2020-07-07: 

Didn't work much on MAFR and just got news that I fucked up my 2nd year. So yeah, might as well continue. 
Since the rest doesn't seem to work out.

I had an important realization: "Why should the server determine the structure and all that?"
There's no real reason to, as the client should do the heavy lifting, and the server just provides what 
the client can't  access. 

So the new plan is to make the server have calls to required resources. As such:
* GET the \<ul> mapping for the Folder Structure. 
* GET the content Section html
* HEAD the pages and return the cookies.
* POST the download requests and return the input stream.   

of course, the client will probably use POST for all these requests for the calls
but maybe it could just be all GET methods. 

With a few parameters:
* CourseId
* SectionId


#### The actual structure
So, 

The Server will have the following calls
* GET /structure | gets the structure html for the client to parse to folders (params: CourseId)
* GET /content  | gets the section content (params: CourseId, SectionId)
* POST /download | downloads and returns the inputStream (params: body: downloadLink)
* POST /visit | needs a few filters to make sure people don't use it to visit weird pages and info pages on the 
account. It enables looking at certain pages required to find download links. 
* GET /peek 
 
#### A quick note on downloading
Moodle downloads usually happen through a php script, something HtmlUnit picks up on, but I haven't seen 
Jsoup getting that right. Jsoup is not meant for what the browser does

#### The enrolment structure
Another problem with the server is that different students use it for different courses. 

The first check can be done with Jsoup. The body id must be `page-enrol-index`. If that is the case,
we can 

There are 4 possibilities:  
* You can enroll without a key
* You can enroll with a key
* You cannot enroll
* course does not exist

I need to find the cues that indicate which of the three is applicable. But in general,
bullet 2, 3 and 4 are the same error in this scope.  

I see a button with class=continuebutton in bullet 3 and 4

bullet 1 has a button with "Enrol me" as value, but so does bullet 2

I can check if the Continue button exist OR the key input field exitst.
If one of those conditions are met, then throw the error that we can't access the course.

so the first check (with jsoup)

2020-06-14:

I made my first Jsoup enrollment post. feels good. Might be able to phase out HtmlUnit all together one day. 

Potential problem: Someone asks structure, need relog **and** not enrolled. What happens? check twice? 
fixed it now, the checks are done seperately. 

#### The Client layout
Never thought I'd finish the server side, but I kinda did. 

So the client is what connects to the server and asks for stuff. 

User wants to Connect to a server, download their course, and keep it up to date.
A user might want to download/update multiple courses. 

So 1 text box for the server url and 1 for password


## Docker
Wtf is the big deal about this shit? Never understood its goal, but fuck it. A lot of employers pay you
if you understand how to work with it. So I will attempt to get the hang of it. 

I torrented the "Docker zero to Hero" course. It's given by some Spanish devops dude. It has subtitles, so I 
don't care, an Indian could give it for all I care.   

There are 7 or so chapters. I'm going to watch each chapter and attempt to document what I learn from it here.

So docker enables you to deploy shit from containers, from what I understand, it means you can containerize
something and make use of it somewhere else as long as it has docker.

Seems cool, I could make some shit and put it in a container with dependencies and give it as an image(?).

I've heard it's more DevOps and less programming, but fuck it. No one seems to give a fuck about
the shit I create, so might as well see if this can make some shit possible.

### Chapter 1 Introduction
#### 1.1 What is Docker?
Jeez, the H sound comes out of his throat like a Russian, хелло, энд велкам. 

He says Docker is a tool to create, deploy and run applications by using containers. 
The container is 1 package with all dependencies and shit. 

He shows himself ssh'ing into some server and deploying a docker image. Then destroys it soon after. 
Seems cool, instant deployment and instant removal, but doubt his image is anything heavy. 

He performs this line: 

`docker run -d -p 1010:80 nginx:alpine`

-d I guess means deploy and -p specifies port of the server. nginx is a webserver so I guess that is what
he deploys. 

He proves that it is running by typing in this:

`docker ps -l`

`-l` I assume means list them, but allah knows what `ps` means.

He removes it with this

`docker rm -fv id` 

the id was some CONTAINER ID he viewed with his list command. 
Don't know what `-fv` means, but fuck it, he'll tell me eventually

End of paragraph.

#### 1.2 What is a container?
He's gonna paint, oh boy.

He says a Container is just a process which is isolated (in a sandbox) in what they call a **namespace**.

**namespace** is the id or the name of the process. 

Then he talks about something called **C groups**, they allow you to limit the resources like cpu power and such.
He says limit, but probably means control. Same thing I guess, but limiting a resource sounds weird to me 
if you try to spin it as a positive.
 
Here's the image he drew:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/Container1.2.png" width="500" height="350">

end of paragraph

#### 1.3 What is an Image?
He recycles picture above ^

He says an image consists of layers. 
Every image starts with something he calls **Scratch**.

On top of it comes the operating system? He shows CentOs as the next layer so I assume that means he 
uses the libraries and other files from CentOs as a basis for his image?

In the next layer he puts in a process, in this case some Apache stuff. 

He calls this a Parent-Child relationship. A child can be a parent of another child who can also be a parent
and so on.

He puts up a fourth layer as a child of Apache, something called cache, probably just an example. 

Here's the image he drew:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/1.3ImageLayers.png" width="500" height="350">


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


   






