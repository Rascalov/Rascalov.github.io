---
layout: post
title:  "Moodle Automatic File Retriever!"
date:   2020-8-10 09:59:21 +0200
categories: moodle java javafx scraper
toc: true
tags:
 - java
 - moodle
 - javafx
 - scraper
---

# Moodle Automatic File Retriever (MAFR)

2020-06-30

I realized that no one will ever use my console app. No matter what source code I hand over and what I might say, 
people are not comfortable inserting their credentials. 

I keep reminding myself of this every time I look at MAFR. It's the only thing I made that I felt kinda good about. Doing a GTAV mod was cute,
but nothing special. MAFR could be of use to a lot of future students of Inholland. If only I would polish it.

So I will, but there are challenges.

## What needs to be done?
A definite answer I do not have, but I have theories:

* Remove the login requirement
* Create a graphical interface
* Optimize the file acquisition proces 

Removing the login is what I believe will help directly. 
This does beg the question: "How the fuck would anything work then?"
I cannot just give session cookies to clients from my own login. Any person could just put those credentials into their browser
and fuck my account up.  

From what I read, this is a situation where the use of a **server-client infrastructure** is warranted.  

### Server
The server has the credentials. Users can set up their own or use the one I set up.
Server is responsible for:
* Login
* Determining and returning the folder structure
* Return content of a section
* Handle and download requested links

### Client
The Client's controlled by the end-user. It'll communicate with the server through api calls. 
Client must enter the server url and the password of the server. Which the server host should give. 
Apart from that, the client's only purpose is to send requests handle the responses given by the server. 

### A realization
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

## What I use in this project
I use the following libraries in particular:
* HTMLUnit: A headless webbrowser library with javascript support. 
* Jsoup: A HTML parser used for web scraping.
 


## What does MAFR do again?
In order to figure out what the **Server's** and **Client's** responsibilities are, I need to recap to myself the idea of MAFR.

1. Login
2. Check access to the given course, enroll if possible
3. Determine structure of the course
4. Index files per section
5. Download files

Updating follows more or less the same steps, except it checks the last-modified date before downloading and remains on 
loop. 

### 1. Login
This is a server-side problem. As the user should not log in.

I created a `MoodleCredentials` class. This class contained a static Map to all the cookies that are received
when logging in. And methods that could be called to check if the credentials were still valid and if a re-log 
is warranted.

The owner of the server should have a `authConfig.xml` containing login credentials and other options like:
* Enabling/disabling enrolling on courses.
* Setting a password the client should use to communicate with the server.

The xml file:
{% highlight xml linenos %}
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
{% endhighlight xml %}

I do the login with HTMLUnit, since moodle is keen on redirecting you everywhere while logging in, it was the 
easiest approach.

{% highlight java linenos %}
{% raw %}
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
{% endraw %}
{% endhighlight xml %}

#### 1.1 Microsoft bot detection
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

{% highlight java linenos %}
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
{% endhighlight java%}

  
### 2. Moodle Courses and enrolling
If you want people to be able to download their courses, there is no doubt you need to allow the server to enroll on
your behalf.

At first, I figured I should let `HtmlUnit` do the heavy lifting and click my way through the enrollment process. 
But that is way too slow, logging in already takes up to 20 seconds. I can't make enrollment take more than 1-2 seconds. 

So I decided to try my hand at submitting the form through `Jsoup`.

#### 2.1 Determine enrollment possibility
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

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/EnrollEligible.png" width="700" height="350" alt="Course Enrollment eligible">

I see a button with `class=continuebutton` in nr. 3 and 4

Nr. 1 has a button with "Enrol me" as value, but so does Nr. 2
I can check if the Continue button exists, OR the key input field exists.
If one of those conditions have been met, then throw the error that we can't access the course.

#### 2.2 Enrolling 
To find out how to send the form to enroll, I need to inspect the values 
that are given to the form once I click submit. 

This is an image of the form element with the hidden inputs:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/EnrollmentElements.png" alt="Form Elements">

So we need to incorporate these values into our request for enrollment. 

I did the request in `Jsoup`, but `HttpURLConnection` should be fine as well.

````
Jsoup.connect("https://moodle.inholland.nl/enrol/index.php")
    .method(Connection.Method.POST)
    .cookies(MoodleCredentials.sessionCookies)
    .followRedirects(true)
    .userAgent("Mozilla")
    .timeout(3000)
    .data("id", doc.selectFirst("input[name=id]").val())
    .data("instance", doc.selectFirst("input[name=instance]").val())
    .data("sesskey", MoodleCredentials.sessKey)
    .data(element.select("input[type=hidden]").get(3).attr("name"), element.select("input[type=hidden]").get(3).val())
    .data("mform_isexpanded_id_selfheader", "1")
    .data("submitbutton", "Enrol+me")
    .post();
````
The result is a fast enrollment process between 0.5 and 1.5 seconds. 

You can also just use `.execute` as long as you set the `Connection.Method` as `POST`.


### 3. Determine structure of the Course
Without the structure, you might as well keep the files on moodle.

I will write about the 2 types of courses used and how to determine the folder structure. 

### 3.1 Two types of courses

My first console app for moodle assumed that only 1 course structure existed:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/moodleStructure.png" alt="Course structure 1">

One menu, with folders. 

But, looking into other courses from other moodle categories, I discovered another structure:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/ModernStructure.png" alt="Course structure 1">

A menu on top, each menu item has its own side menu. In my opinion this structure is superior to the former. 
To think the IT category still uses the former structure. 

Anyhow, I started with dissecting the menu each one has.

#### 3.2 Menu structure

This image explains most of it:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/moodleStructure.png" width="700" height="350">

Then, once you select a course subject, it opens the submenus:

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/moodleStructure2.png" width="700" height="350">

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
If more levels are present, I loop the method until all levels have been accounted for. 

##### 3.2.1 Some code 
This is all client side, as the server only needs to return the html of the course page once!

````
private MoodleFolder getLiteralStructure(MoodleFolder mainfolder, Element courseSubjects){
    // get all the folders of the menu
    var children = courseSubjects.children();
    for (Element subject : children ){ // loop over them to get data-section and name
        var subjectElement = subject.select(".mt-heading-btn").get(0);
        MoodleFolder subjectFolder = new MoodleFolder(subjectElement.text(), Integer.parseInt(subjectElement.attr("data-section")));
        System.out.println(subjectFolder.getName() + " Is the Subject Folder");
        if(subject.children().size() > 1){
            // if true, we use this method again to determine additional folders.
            var subjectBody =  subject.child(1).child(0).child(0);
            subjectFolder.getMoodleFolders().addAll(getSubfolders(subjectBody));
        }
        mainfolder.getMoodleFolders().add(subjectFolder);
        System.out.println("-----------------------------------------------------------------------------");
    }
    return mainfolder;
}
List<MoodleFolder> getSubfolders(Element sublevelList){
    // Handle additional subfolders and, sometimes, subfolders within subfolders (aka multi-level folders).
    List<MoodleFolder> moodleFolders = new ArrayList<>();
    MoodleFolder folder = null; // Matriarch folder in case of multi-level folders
    for(Element section : sublevelList.children()){
        if(section.is("a")){
            folder = new MoodleFolder(section.text(), Integer.parseInt(section.attr("data-section")));
            moodleFolders.add(folder);
        }
        else if(section.is("div"))
            // Sub folder has a subfolder, so repeat the method.
            moodleFolders.get(moodleFolders.indexOf(folder)).getMoodleFolders().addAll(getSubfolders(section));
    }
    return moodleFolders;
}

````

#### 3.3 Handling the second structure
This structure contains a row of buttons that each have their own menu.

I will call these buttons "Row-Buttons". I do prize myself for my creative naming. 

I need to make a "wrapper" that first checks for this structure and if it is found, handle it.

The way they connect these menus with the Row-Buttons, is with a `data-pane` attribute value. Which is a number. 
I might be able to use this.

I can get each Row-Button's `data-pane` value, and then, in the folder of that Row-Button, I will invoke my already 
constructed methods to index that connected mt-menu.

Like so:
````
if(doc.selectFirst("ul.mt-topmenu") != null){
    for (Element menuOption : doc.selectFirst("ul.mt-topmenu").select("a")){
        String folderName = menuOption.selectFirst("span.mt-btn-text").text();
        String dataPane = menuOption.attr("data-pane");
        Element mtMenu = doc.selectFirst("div[data-pane=" + dataPane + "]");
        MoodleFolder menuFolder = new MoodleFolder(folderName, Integer.parseInt(menuOption.attr("data-section")));
        if(mtMenu.select("div.mt-sidemenu") != null) // if it has a menu, use the method I showed in 3.2
            menuFolder = getLiteralStructure(menuFolder, mtMenu.selectFirst("div.mt-sitemenus"));
        mainFolder.getMoodleFolders().add(menuFolder);
    }
}
````
With this, I can determine the structure of 99% of all courses a student can access on the Inholland Moodle environment.

### 4. Index Files
How the server requests the content per section you can see in the github. It uses the `data-section` attributes
gathered while structuring.

What matters is how the Client will handle the HTML and look through it to find the files.

The File a teacher uploads to moodle is usually characterized by something called a `mod-type`
There are a lot of mod-types, really, a lot.

To name a few:
    
    modtype_resource: holds one(?) link to a resource
    modtype_label: holds text without resource links, though could include links. Usually used for Header Titles
    modtype_book: Holds a video(?) in a seperate window
    modtype_folder: Holds a folder which can include multiple files, in multiple trees even
    modtype_forum: Holds a forum which has multiple announcements from the course
    modtype_assign: Holds an assignment field where users can drop assignments, probably never interesting
    modtype_page: Holds a text file, some have a Last-Modified in string format
    modtype_url: Holds url should be saved as a textfile
    modtype_wiki: Holds a page where Groups can create their own wikipedia like page.
    modtype_feedback: Holds a feedback link to a document you uploaded for review.
    modtype_groupselect: Holds a list of group members, can be saved as textfile
There are more, but I have yet to determine what their usages are

This part frustrated me for quite some time. I don't want to make a case for every `mod-type` and download them all 
in a different way. Too much work for so little. Most students only have concern for resource files like pdf, 
word files and so on. Which 80% of the time is held within `modtype_resource`

Text files are also annoying, as they must be indexed and downloaded differently, and could not be updatable with 
a Last-Modified header.

So I came up with a solution that would save me some work.

#### 4.1 The snapshot.html
The idea is simple, all the content from a section will be saved as an HTML file.
Text will keep its html formatting and all the mod-types will remain accesible throught the snapshot. 

For example, a section with only Assignment mod-types will do no good when saved as files. But with the snapshot,
you can look at them and go to the assignment link whenever desirable. Without it being a nuisance as a file.

Usually, a student has been logged in to Moodle anyway while using this program, so they can visit the links 
in the snapshots without an issue.  

#### 4.2 Indexing resource files
The idea of indexing the files is to put each discovered file into a list which will be downloaded at the end. 

First things first, images and videos I consider "exceptional content" as it can be embedded into the section without
its own page.

So I check for those kinds of files first like so:

````
private List<MoodleFile> getExceptionalContent(Document doc) {
    // It has been proven that images and videos can be embedded in multiple, if not all, mod-types.
    // No doubt there will be more resources that will fit this criteria,
    // therefore the method name is very generalized.
    var moodleFiles = new ArrayList<MoodleFile>();
    for(var video : doc.select("video")){
        String url = video.selectFirst("source").attr("src");
        moodleFiles.add(getFile(peek(url), url));
    }
    for (var image : doc.select("img[src^=https://moodle.inholland.nl/pluginfile.php]")){
        String url = image.attr("src");
        moodleFiles.add(getFile(peek(url), url));
    }
    return moodleFiles;
}
````
This is used in the section, as well as in the page mod-type and (rarely) in resource mod-types. 

Resource files will be obtained by visiting their pages and determining the resource url from a number of
places that I have observed contain those resource links.

````
private MoodleFolder getPotentialResourceFiles(Element li, MoodleFolder folder) {
    String link = li.selectFirst("a").attr("href");
    var peekDoc = peek(link);
    MoodleFile resourceFile = null;
    if(peekDoc.get("Content-Disposition")!=null) // if true, it is downloadable
        resourceFile = getFile(peekDoc, link);
    else {
        Document doc = visit(link + "&forceview=1");
        var workaround = doc.selectFirst("div.resourceworkaround");
        var iframeResource = doc.selectFirst("iframe#resourceobject");
        var objectResource = doc.selectFirst("object#resourceobject");
        if(workaround != null)
            resourceFile = getFile(peek(workaround.selectFirst("a").attr("href")),workaround.selectFirst("a").attr("href"));
        else if(iframeResource != null)
            resourceFile = getFile(peek(iframeResource.attr("src")),iframeResource.attr("src"));
        else if(objectResource != null)
            resourceFile = getFile(peek(objectResource.attr("data")),objectResource.attr("data"));
        else {
            // I discovered resource files with no files but rather images and text
            Element main = doc.selectFirst("div[role=main]");
            folder.getMoodleFiles().addAll(getExceptionalContent(Jsoup.parse(main.html())));
            resourceFile = new MoodleTextFile(main.html(),"");
            resourceFile.setName(main.selectFirst("h2").text());
        }
    }
    if(resourceFile != null){
        folder.getMoodleFiles().add(resourceFile);
    }
    else {
        System.out.println("A null file was given, ignoring...");
    }
    return folder;
}
````

#### 4.3 Indexing Folder mod-types

Folder mod-types are weird, I don't know why it exists. It's basically a glorified subject folder.  

Nonetheless, it has two forms:

1: Own page

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/folderOwnPage.png">

2: Embedded onto the section

<img src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/EmbeddedFolder.png">

Folder pages can have images, videos, text and all other kinds of stuff.
So I need to use reuse the `getExceptionalContent()` method and create a snapshot.

I have only ever seen folders that look like the image above. So I won't bother
creating a folder structure for it apart from the initial folder.

As for the files, they seem to always be `<a>` tags without a class.
So I'll use the css selector `:not([class])` the index them.

The result:

````
private MoodleFolder getPotentialFolderFiles(Element li, MoodleFolder folder) {
    List<MoodleFile> folderFiles = new ArrayList<>();
    String folderName;
    Element folderMain;
    // check if either link or embedded, which affect the folder name and element to take files and snapshot from.
    if(li.selectFirst("a[href^=https://moodle.inholland.nl/mod/folder/view]") == null){
        folderName = li.selectFirst("span.fp-filename").text();
        folderMain = li;
    }else{
        folderMain = visit(li.selectFirst("a").attr("href")).selectFirst("div[role=main]");
        folderName = folderMain.selectFirst("h2").text();
    }
    var snapshotFile = new MoodleTextFile(folderMain.html(), "");
    snapshotFile.setName(folderName.replaceAll("[^a-zA-Z0-9\\.\\-]", "-"));
    folderFiles.add(snapshotFile);
    for (var downloadLink : folderMain.selectFirst("div.filemanager").select("a:not([class])")){
        folderFiles.add(getFile(peek(downloadLink.attr("href")), downloadLink.attr("href")));
    }
    MoodleFolder folderMod = new MoodleFolder(folderName);
    // user folderMod.getName instead of folderName, because the prior is sanitized.
    new File(folder.getPath() + "/" + folderMod.getName()).mkdir();
    folderMod.getMoodleFiles().addAll(folderFiles);
    folderMod.setPath(folder.getPath() + "/" + folderMod.getName());
    folderMod.setSnapshotable(false);
    folder.getMoodleFolders().add(folderMod);
    return folder;
}
````



Notice that all file indexing methods use the `getFile()` method along with `peek()` 

`peek()` is the method used to request the server to in turn make a HEAD request to moodle for a particular file or page.
`peek()` then returns the received headers. Which will be used by `getFile()` to get the filename and
Last-Modified header.


### 5. Download Files
The final part, I had thought this to be the worst part, as you have to download the 
files on the server first and then transfer them over to the client.

However, it seems to be functioning just fine. The heroku server has been deployed in europe. 
And the results are better than I could have asked for.

Now downloading is very straightforward: Get the InputStream from the server and transfer the bytes to a new
`FileOutputStream`   

````
public void downloadFile(MoodleFile file, String path){
    // get jsoup response and download file.
    if(file instanceof MoodleTextFile) // but if it's a text/html file, let it get the string bytes
        ((MoodleTextFile)file).download(path);
    else {
        String dl = file.getDownloadLink();
        try {
            dl = URLEncoder.encode(dl, StandardCharsets.UTF_8.toString());
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        Connection.Response response = null;
        try {
            response = Jsoup.connect(serverUrl+"/download")
                    .header("moodleServerAuth", serverPassword)
                    .header("downloadLink", dl)
                    .method(Connection.Method.POST)
                    .maxBodySize(0) // 0 = no size limit
                    .execute();
        }catch (IOException e){
            if(e instanceof HttpStatusException){
                HttpStatusException hse = (HttpStatusException) e;
                System.out.println(hse.getStatusCode());
            }
            System.out.println("File that caused error: "+file.getDownloadLink());
            throw new RuntimeException(e.getMessage());
        }
        try {
            response.bodyStream().transferTo(new FileOutputStream(path + "/" + file.getName()));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
````
That's about it for the backend. 
I could go into the model classes, but if you're interested in that,
you might as well visit the github. 

Most of this work is far from perfect. Improvements will be made in time. But for now,
Moodle Automatic File Retriever is functional. 

Next up is the client GUI. 
 

### The Client layout

The client's UI is still up for debate, even though it is probably going to be very straightforward and (decently) clean.

A few logos, some input and you got half the appeal and functionality. 

I'll make the buttons into Inholland's moodle buttons. For at least a bit of familiarity.

User wants to Connect to a server, download their course, and keep it up to date.
A user might want to download/update multiple courses. 

I also want the client to save what the user set up to update and download when closing.
That would make it easier for the user to open Mafr and resume updating courses. 

So I figured something out, and made it look a bit similar to the Inholland Moodle stuff:

<img alt="Client Layout" src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/ClientLayout.png">

This design should cover all that I want to do on the front end. Can't be that confusing to use
either.

And as bonus, I made a shitty edit to the Moodle icon with a scraper:

<img alt="Moodle Scraper Icon" src="https://raw.githubusercontent.com/Rascalov/Rascalov.github.io/master/images/MoodleScraperIcon.png">

### A certificate issue
On the 21st of September 2020, I encountered an SSL handshake issue that ruined the server
connection to moodle. From what I gather, Moodle renewed/updated their certificate and
both htmlunit and Jsoup could not update their end. 

HtmlUnit has an option to disable SSL related problem, but Jsoup does not. 
I ended up using a static initialize method to make sure all certificates were to be 
allowed. This, I consider a workaround, not a solution. But it worked all the same. 

