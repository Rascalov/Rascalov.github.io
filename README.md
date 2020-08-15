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

I realized that no one will ever use my console app. No matter what source code I hand over and what I might say, 
people are not comfortable inserting their credentials. 

I keep reminding myself of this every time I look at MAFR. It's the only thing I made that I felt kinda good about. Doing a GTAV mod was cute,
but nothing special. MAFR could be of use to a lot of future students of Inholland. If only I would polish it.

So I will, but there are challenges.

### What needs to be done?
A definite answer I do not have, but I have theories:

* Remove the login requirement
* Create a graphical interface
* Optimize the file acquisition proces 

Removing the login is what I believe will help directly. 
This does beg the question: "How the fuck would anything work then?"
I cannot just give session cookies to clients from my own login. Any person could just put those credentials into their browser
and fuck my account up.  

From what I read, this is a situation where the use of a **server-client infrastructure** is warranted.  

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

#### A realization
I had an important realization: "Why should the server determine the structure and all that?"
There's no real reason to, as the client should do the heavy lifting, and the server just provides what 
the client can't  access. 

So the new plan is to make the server have calls to required resources. As such:
* GET the \<ul> mapping for the Folder Structure. 
* GET the content Section html
* HEAD the pages and return the cookies.
* POST the download requests and return the input stream.   

of course, the client will probably use GET and POST for all these requests for the calls
but maybe it could just be all GET methods. 

With a few parameters:
* CourseId
* SectionId
* password header

The Server will have the following calls
* GET /structure | gets the structure html for the client to parse to folders (params: CourseId)
* GET /content  | gets the section content (params: CourseId, SectionId)
* POST /download | downloads and returns the inputStream (params: body: downloadLink)
* POST /visit | needs a few filters to make sure people don't use it to visit weird pages and info pages on the 
account. It enables looking at certain pages required to find download links. 
* GET /peek | tell the server to do a HEAD request on a link. Used for potential files. 

### What I use in this project
I use the following libraries in particular:
* HTMLUnit: A headless webbrowser library with javascript support. 
* Jsoup: A HTML parser used for web scraping.
 


### What does MAFR do again?
In order to figure out what the **Server's** and **Client's** responsibilities are, I need to recap to myself the idea of MAFR.

1. Login
2. Check access to the given course, enroll if possible
3. Determine structure of the course
4. Index files per section
5. Download files

Updating follows more or less the same steps, except it checks the last-modified date before downloading and remains on 
loop. 

#### 1. Login
This is a server-side problem. As the user should not log in.

I created a `MoodleCredentials` class. This class contained a static Map to all the cookies that are received
when logging in. And methods that could be called to check if the credentials were still valid and if a re-log 
is warranted.

The owner of the server should have a `authConfig.xml` containing login credentials and other options like:
* Enabling/disabling enrolling on courses.
* Setting a password the client should use to communicate with the server.

The xml file:
````    
<ServerConfig>
    <LoginCredentials>
        <Username>yourEmail@student.inholland.nl</Username>
        <Password>yourPassword</Password>
    </LoginCredentials>
    <ServerAuth>
        <ServerPassword>theServerPassword</ServerPassword>
    </ServerAuth>
    <AlllowEnrollment>true</AlllowEnrollment>
</ServerConfig>
````

I do the login with HTMLUnit, since moodle is keen on redirecting you everywhere while logging in, it was the 
easiest approach.
````
page = webClient.getPage(loginUrl);

HtmlForm form = page.getForms().get(0);
HtmlEmailInput uname =  form.getInputByName("UserName");
HtmlPasswordInput pass =  form.getInputByName("Password");
HtmlElement buttonElement = form.getElementsByTagName("span").get(1);
uname.setValueAttribute(usr);
pass.setValueAttribute(pwd);
page = buttonElement.click();
DomElement BypassButton = page.createElement("button");
BypassButton.setAttribute("type", "submit");
HtmlForm formBypass = page.getForms().get(0);
formBypass.appendChild(BypassButton);
page = BypassButton.click();

````

##### 1.1 Microsoft bot detection
As you are logging in with the HTMLUnit `WebClient`, you notice something about a `BypassButton`. 

If Microsoft suspects a bot to have logged in to its services, they will first redirect the bot
to a page where they need to click a button within `<noscript>` tags. Which a bot cannot click.

I found 2 ways to get passed this:
1. Replicate the form and its input values and send it over a `WebRequest` then copy the resulting moodle cookies
2. Create a button in the form that has the same characteristics as the `<noscript>` button, and click it. 

From the code, you can conclude I went with option 2. There will come where it will fail, but I will then use option 1.
  
##### 1.2 Cookies and session key
There are two things that moodle checks to see if you are a logged in user.
* Cookies
* Session key (sessKey)

Cookies should be received when successfully logged in.

The session key can be taken from the logout button on the moodle home screen. I haven't seen it come with the cookies.

I use a `HashMap`, and a long `String` to store the cookies.
Jsoup requires a Map, while `HttpURLConnection` requires a String seperated by `;`

```` 
if(page.getUrl().toString().equals(succesPage)) {
    // if successful, we can grab the generated sessKey from the dynamic logout link
    DomElement link = page.querySelector("a[aria-labelledby=actionmenuaction-6]");
    String sesskeyString = link.getAttribute("href").split("\\?")[1];
    sessKey = sesskeyString.split("=")[1];
    sessionCookies = new HashMap<>();
    webClient.getCookieManager().getCookies().forEach(c -> sessionCookies.put(c.getName(), c.getValue()));
    StringBuilder sbCookies = new StringBuilder("");
    MoodleCredentials.sessionCookies.forEach((k, v) -> {
        sbCookies.append(k + '=');
        sbCookies.append(v + ';');
    });
    sessionCookiesString = sbCookies.toString();
    System.out.println("Logged in, cookies and sessKey Stored");
    System.out.println(sessionCookies);
    success = true;
}
````

  
#### 2. Moodle Courses and enrolling
If you want people to be able to download their courses, there is no doubt you need to allow the server to enroll on
your behalf.

At first, I figured I should let `HtmlUnit` do the heavy lifting and click my way through the enrollment process. 
But that is way too slow, logging in already takes up to 20 seconds. I can't make enrollment take more than 1-2 seconds. 

So I decided to try my hand at submitting the form through `Jsoup`.

##### 2.1 Determine enrollment possibility
When you are not enrolled to a course, it will direct you a special page when attempting to visit said course. 

The first check can be done with Jsoup. The body id must be `page-enrol-index`. If that is the case,
we can check whether we can enroll or not. 

There are 4 possibilities:  
1. You can enroll without a key
2. You can enroll with a key
3. You cannot enroll
4. course does not exist

I do not plan to have people insert their enrollment key to enroll into a course. So only nr. 1 will be 
eligible to enroll. 

An example of nr. 1:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/EnrollEligible.png" width="500" height="350" alt="Course Enrollment eligible">

I see a button with `class=continuebutton` in nr. 3 and 4

Nr. 1 has a button with "Enrol me" as value, but so does Nr. 2
I can check if the Continue button exists, OR the key input field exists.
If one of those conditions have been met, then throw the error that we can't access the course.

##### 2.2 Enrolling 
To find out how to send the form to enroll, I need to inspect the values 
that are given to the form once I click submit. 

This is an image of the form element with the hidden inputs:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/EnrollmentElements.png" alt="Form Elements">




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

I see a button with `class=continuebutton` in bullet 3 and 4

bullet 1 has a button with "Enrol me" as value, but so does bullet 2

I can check if the Continue button exists, OR the key input field exists.
If one of those conditions have been met, then throw the error that we can't access the course.

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

I also want the client to save what the user set up to update and download when closing.
That would make it easier for the user to open Mafr and resume updating courses. 

So I figured something out, and made it look a bit similar to the Inholland Moodle stuff:

<img>

And here's the little icon:

<img>


This design should cover all that I want to do on the front end. 

#### The new indexing of files.
I have been thinking about this since I made mafr 1.5. The current way of files is
very mediocre. I merely check all `<a>` links and if they are downloadable, I download them.

There are a few downsides to it that make it unsustainable:
* Videos can be embedded without `<a>` tag
* images can just be `<img source=>` without `<a>` tag
* Some `<a>` links have text underneath that must be caught as well.

These are the 3 I discovered, as I look at more courses, I'll probably find more downsides.

As I said on the github, Text is a unique challenge on its own, as it is not a file, but merely a big string, 
it has no Last-Modified or other HTTP headers. Unlike videos and Images.

I can update based on text, if the new one does not equal the text of the older one. Might work, might not.
It will depend on how I name the text files. 
Which is another problem. I'll talk about that later.

##### Mod-type based indexing
from my github:

    modtype_resource: holds one(?) link to a resource
    modtype_label: holds text without resource links, though could include links. Usually used for Header Titles
    modtype_book: Holds a video(?) in a seperate window
    modtype_folder: Holds a folder which can include multiple files, in multiple trees even
    modtype_forum: Holds a forum which has multiple announcements from the course
    modtype_assign: Holds an assignment field where users can drop assignments, probably never interesting
    modtype_page: Holds a text file, some have a Last-Modified in string format
    modtype_url: Holds url should be saved as a textfile
    modtype_wiki: Holds a page where Groups can create their own wikipedia like page.
    modtype_groupselect: Holds a list of group members, can be saved as textfile
    
    Every mod type can have text, no matter what they convey (save for modtype_label, that's its entire purpose)
    
    For every li there should be a text file, the text file would have the name of either the resource it is associated with OR, in case it has none of that, its first p tag textcontent will be the filename.
    
    After a Title (h3), its possible for there to be a "summary" class, which contains text about the folder. The title of these textfiles should be equal to the title of which they summarize (so, the h3 tag).
    
    modtype_label has a class "contentwithoutlink", which holds all the text. You need to get all the text, but also look voor ahref tags, if they embed them on a word, you have to seperate that, if not, it will get stored in the text file without worries.
    
    Other modtypes have divs that hold the text before (NOT SEEN YET, BUT COULD EXIST) and after the resource link.
    
    The after class is "contentafterlink", its title should be the "TEXT" + resource_name text value.

So I have multiple types to handle. 
Should I do this in classes? Should I do this in one big Switch statement? I do not know yet.

Some mod-types require the making of new folders and indexing the files and text in there. 

I could just make methods within the same class. With a big switch statement. 

#### You know what
fuck it, the text problem can be solved by saving the entire content section as a html file. 

All formatting will be kept and all embedded information is accessible.
I will call it a "snapshot". Individual label mod-types will still be downloaded as text.

#### Indexing images and videos (and other irregular embeds)
As there are elements that are exclusively embedded onto the section, I made a method that handles images
and videos. Images are on every section in the form of icons. Which I need to avoid. 

Currently, I loop over the images and if they have no class attribute, it is to be downloaded. 
I am looking for a css selector that selects all images with no class attribute. 
That would really speed up the indexing. 

Alright, 5 minutes later I figured out css selectors have a `:not()` option.
So I can ignore the class attribute with `img:not([class])`. Pretty neat. 

However, I have seen images in resource files that have the class "resource"
So I have to edit the selector to `img[src^=https://moodle.inholland.nl/pluginfile.php]`
This ensures I get every image that is a plugin file and not a theme file like the icons. 

#### indexing mod-type folder files
the folder mod type is a special map where you can put files. They can be either embedded into the section, or have their
own page. 

This is a folder with its own page:

<img alt="image folder own page">

And this is a folder embedded to the section:

<img alt="image folder embedded">

In both cases, a new folder should be made equal to the name of the folder.

There is a case where there are multiple folders, but I lost the course which made use of that. 



#### The two types of Moodle structure
Since I am an IT student, my courses look like this:

<img alt="image of It course with no Row-Buttons">

But when I shopped for moodle courses that could help me find errors in my software,
I noticed a lot of courses structured themselves like this: 

<img alt="image of course with Row-Buttons">

I will call these buttons "Row-Buttons". I do prize myself for my creative naming. 

The second one seems to be the original structure, as it incorporates the structure I am familiar with.

I need to make a "wrapper" that first checks for this structure and if it is found, handle it.

Multiple issues. The way to get the content section is the same. Which is shitty because the mt-menu (the navigation menu) does not 
come with the request for that section. Instead, it is hidden until its associated button is clicked.

The way they connect these, is with a `data-pane` attribute value. Which is a number. I might be able to use this.
I can get each Row-Button's `data-pane` value, and then, in the folder of that Row-Button, I will invoke my already 
constructed methods to index that connected mt-menu.

I will make some code for that later, first comes the mod folder. 



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

end of paragraph

#### 1.4 Containers vs virtual machines
He shows an image with showing container and vm properties. 

Virtual machines each have their own guest os. So resources are saved with 
containers. He claims the best thing about docker is that you could destroy the containers instantly.

Here's the image he referred to:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/1.4Comparison.png" width="500" height="350">

End of paragraph.

#### 1.5 Basics of a Docker file. 
He refers back to the docker image. According to him, we need to make a docker file.
A docker file is a plain text file where you define the instructions to create your image.

According to him, the file usually starts like this:

FROM operating_system_name
RUN = install the software you want through the selected os' package manager.
CMD = the command to execute after container creation. 

This is all he says currently, more in the future.

The example file he gave:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/1.5DockerFile.png" width="500" height="350">

end of paragraph.

#### 1.6 Docker's architecture

He shows an image of a Docker host?

Docker host is anything that runs a docker server.  
The Docker host contains:

The docker daemon: the process that is started once you set up docker.

REST API: communication to the daemon

CLI: You typing commands to the API. 

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/1.6DockerHost.png" width="500" height="350">

End of paragraph

#### 1.7 Layering in Docker
He shows an image composed by layers. 
The instructions from last video could be considered layers. 
They are read only layers. Instead you can create another layer on top of the 
desired layer. 

If you want to update it, you can create a new image starting from a specific layer.

So you basically version your images. 

He ends with this image:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/1.7Layering.png" width="500" height="350">


So far Chapter 1 is very basic. But that's probably not a bad thing. His accent fucks 
up some words, but it's understandable with the generated subtitles.

End of paragraph.

End of Chapter 1. 

### Chapter 2 Ready, Let's head to the installation. 

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


   






