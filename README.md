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
Het meeste van Mafr verbetering is te vinden op haar github. Ik heb nauwelijks de wil om ermee door te gaan.
Interesse is laag en uitwerking is vanaf het begin al flawed.  
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

<img src="https://github.com/Rascalov/Rascalov.github.io/blob/master/images/index.jpeg" width="400" height="200">


   






