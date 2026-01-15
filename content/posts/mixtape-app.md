+++
title = "Mixtape App"
date = "2023-10-28T02:35:03"
slug = "mixtape-app"
draft = false
+++

A longer project that I have been working on that took around a few months to build out.

{{< github repo="phantasmfour/mixtape_app" showThumbnail=false >}}

Why Build This?
---------------




Two things I was looking to accomplish with this:

* Music app with no Ads and legal music
* See how ChatGPT4 codes with languages other than Python


I have been a fan of Mixtapes since I was younger. They have died off a lot in popularity and it gets greyer and greyer about what is a free mixtape and what is commercial. DatPiff also recently shut their app down so I was looking for an alternative with no ads but sadly that does not exist.

Like everyone I have been playing with ChatGPT and I tried out the code interpreter to do some data analytics and was very impressed with what it was able to do. Since I had a GPT4 subscription I decided it would be good to get some other use out of it and this app has been on my list for a while.

I am not going to dive specifically into the coding that I did since its a large project. I am going to give a rough timeline and the largest issues I ran into. 




First Few Weeks
---------------




My first few weeks was me reintroducing myself into Flutter. I wanted to make the app available in a Web Browser, Android and iOS. I also write most of my code in Linux and XCode sadly is only for Apple devices. And with the state of Mac Virtualization is kind of non-existent for App development on newer versions of iOS.

Flutter is great and GPT4 was able to start working with it very easily. It pointed me to [just\_audio](https://pub.dev/packages/just_audio) which is the core library of the project.

The very first screen we worked on was the Album List Screen

![](/images/mixtape1st.png)Originally this was all written around json files that we loaded in from the webserver. That pointed to the images and the songs within the Albums  
In order to generate those json files I had GPT write a few scripts to take the mp3 files within folders and put this info into a json file.  
This is where I hit my first issue.

### **Metadata**

I don't envy and company that relies on metadata included in files for any information. With Mixtapes depending on where you downloaded them from the metadata would be non-existent or lacking important features.  
In order to populate the title, artist and song name I relied on the metadata within mp3 files.   
GPT wrote a quick script to run through each mp3 and extract the necessary metadata. If it was not there then it prompted me to fill it in which took a good amount of time.   
This took a while to input all this metadata but it is well worth the trouble as you do it once and have it forever. 

After I was able to have all the metadata extracted I then put it into json files. I was then able to move onto the screen you see when you click an Album. 

![](/images/mixtape2nd.png)I kept this really simple.   
The hardest part of this was creating a custom navigation bar so that when you scroll the Album cover and back button go away. Without this the back button was in a navigation bar and my data was below it.

After this is was the Now Playing Screen

![](/images/mixtape3.png)Again kept it simple for the UI. But this is where things start getting more complicated.




HLS Fun
-------




When I was testing the first version of the app I had a large gap between when the first song ended and when the second would start since the next song did not start loading yet.  
With just\_audio you are given the ability to have multiple players. GPT gave me the idea to have an active player and a next song player which loads the next song and switches between the players once the song ends. Then you just repeat the process and the inactive player loads the next song.   
I needed to use provider to manage the state of the players and notify the UI widgets whenever the active player was switched to the next player as flutter does not seem to have a concept of global objects.  
Ultimately this ended up causing more issues and I ended up reverting back to one player. What pushed me back to one player was in the end when I needed to use [just\_audio\_background](https://pub.dev/packages/just_audio_background) there is a limitation of only being able to use one player.  
  
I wanted to have a library that I could import and call upon whenever we needed to change the song or update the data on screen. GPT told me to name the file audio\_service which I thought was a great name. But GPT was confused in doing this as [audio\_service](https://pub.dev/packages/audio_service) is a legitimate library which the same guy(Ryan) that wrote all the just\_audio modules. This also pushed me to switch back to one player as getting player status in your notification bar and lockscreen would have needed me to directly import audio\_service and use players from there. This would have been a larger rewrite of a core process of the app and I decided against it.

This finally brings us to HLS. Since this app would ultimately be running on my iPhone I was forced into HLS as Apple does not support DASH. [HLS](https://developer.apple.com/streaming/) is pretty cool though. just\_audio also supported it very easily. GPT was able to write the code to use ffmpeg to create m3u8 and ts files from the existing mp3's. Besides the time it took to create all the files on the initial run, HLS works great and basically eliminated my need to have multiple players. I was also surprised how easy it is to setup as you basically just create the files and host them somewhere and your done.  
The only issue I ran into with HLS is just\_audio does not support it for browsers so I had to write in a check within my audio\_service to see if the client is on a Web browser and force only mp3's to load.




Database
--------




With adding in HLS, wanting to support playlists, and to get away from the JSON files due to load times it was time to implement a database.  
This was supposed to be a fairly quick project so I went the route of using Google Firebase. I am not super concerned of the privacy implications for this app and its one less thing I have to manage.   
GPT recommended Cloud Firestore and I remembered using it a few years back for one of my first forays with Flutter.  
You can also do your authentication via it and leverage this to protect the database.  
With Firestore as long as you have the apiKey and your database rules are setup poorly anyone can write to your database once they extract the api key from network traffic or the apk. In order to get around this you need to write good database ACLs. I kept mine simple as we only allow reads to the albums from users and the users playlists are only accessible with the matching UID


```
service cloud.firestore {
  match /databases/{database}/documents {
  match /albums/{document=**} {
      allow read: if true; // Allow anyone to read
      allow write: if request.auth != null && request.auth.token.isServiceAccount == true; // Only allow the service account to write
    }
  
    match /users/{userId}/{document=**} {
      // Allow read and write only if the user's UID matches the document's name
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}

```

I had GPT rewrite the scripts we used to load data into JSON into the Cloud Firestore database. 

One thing Firestore does extremely well which I don't fully understand it is caching. If you go offline most of your data is still there. There is not much information around the caching setup that I could find or knobs to tune.

One thing Firestore does extremely poor is querying. You search for exact matches via queries. When I was writing my search I had in mind just fuzzy matching but you cannot achieve this via Firestore.   
To work around this I added a keywords field which tries to pull out some of the song info. Then when you search you are not searching for exact matches but keywords of song titles to try and achieve this fuzzy matching. It works best with Artist and Album names since those are easy to remember and provide an exact match.   
The only other way to do it would have been to keep adding keywords of every possible combination of fuzzy matching for each song, album title, and artist. This most likely would have been too much data stored in the database, but it could have worked.

![](/images/mixtape4.png)


Authentication
--------------




This app is just a music player. I have no need for anyones email or any information about them. But email is a super convient way to just associate a user with an account to store their playlists so they can login on other devices and carry this data over. So I ended up supporting email accounts for this purpose.

However we also have a Skip button. This just does anonymous authentication so you can just get in, play music, and use playlists. However if you do sign out you lose that playlist data since I can no longer associated you.

![](/images/mixtape5.png)I again made this super simple and I think **Skip** should be used as the main option as we don't need any personal information like an email for the basic functionality of the app.  
I probably could have made the Skip button a lot bigger to emphasize my push to people using anonymous accounts but the option is still there.




Finishing Touches
-----------------




### Playlists

Once we had authentication configured and the database setup we were able to start creating and storing playlist data. The setup for this is pretty simple and I copied over most of my code from song\_list\_screen.  


![](/images/mixtape6.png)![](/images/mixtape7.png)Basically playlists are just associated with the user and I am then just treating them like an Album with setting the player to the current queue of songs from the playlist.

### Mini Player

All good music player apps will have a mini player once you start playing a song and leave the now playing screen.  
There was no easy way to do this outside of building a MiniPlayer widget on every screen I wanted it to show on. I was able to house the code in its own module so its really just calling on the widget and forcing it to the bottom.  
I hit a small issue with the custom navigation bars on the song\_list\_screen which caused the mini player to cover the last song of your album. To get around this I just added some padding to the bottom if the mini player is loaded. 

### Downloads

Downloads were pretty fun. I used the Dio library and just stored the songs on the users device. When you tap the download button you end up downloading the mp3 version of the song.

Every time you go to set the queue of the player it will check if the song has been downloaded locally and use the mp3 version on the device.   
If you download the song after you already start playing another song in that album then we will not use that recently downloaded file. I setup a queue every time you press a song so if it was not downloaded then the queue will reference the web URL. 

To build a good download button was a bit harder. I ended up needing to create a custom button. The way I went about this is probably not the best but when you click into an album or playlist I check every single song to see if it has a local file associated with it on your device. If so then show the music icon. If not the download icon. If you click the download button then show a loading icon until the song is downloaded.




How did GPT4 do?
----------------




GPT4 Code Interpreter was really helpful. I would say its greatest benefit was being able to write quick python scripts for managing the server side setup. It would write code in seconds that it would have taken me 10-30mins to get working. 

With Flutter it was also very impressive. It was able to help write usable code. Some of it was dated as new modules are developed and things are deprecated.   
I also cannot judge how well it was doing since I am not that familiar with Flutter and have not used it for a few years.

It had some road bumps with the naming of the audio\_service and suggesting things that just would not work.  
However it is good once you get a better understand of how to use it.   
It struggled grasping the entire project as it got bigger and bigger. But if I gave it specific tasks to write a specific widget it still excelled but relied on me a bit more for implementation. 

Error handling seemed to be hit or miss. With such a large project when it wrote in an error it sometimes would not be able to fix it on its own. Most of the time however the code produced did not have any errors.

Asking it questions is great but at some point documentation just becomes easier when you are getting the wrong answers or not the best one. This is mostly a gripe when having it give me information on a specific package like just\_audio. The documentation in this case is really good and I ended up referencing it more than asking GPT.

Can you work with it to build you an entire app? *Yea*  
Would I have been able to do it without external documentation? *Maybe*  
Is it at the point where someone who does not know how to code to make an app? *No*  
Is it at the point where someone who does not know a specific programming language can still use it? *Yes*

Overall its amazing and it significantly sped up the process of me coding this app and that is why I used it.   
What I have done in the past is just looked up YouTube tutorials of similar apps and basically reused code into mine.   
I think using GPT here is better than that and it allows you to ask questions about the code to get an even greater understanding.  
I would definitely use it again.   
I also think it has a lot of potential for the future when it keeps improving. The answers to my questions above might change.  
 




Conclusion
----------




There are a multitude of things I could have added:


* Queue that you can manage of upcoming songs
* Deleting Downloading Songs
* Downloading Images for Songs/Albums when Offline
* Looping for song controls
* Multiple Players
* Mixtape Request Integration
* Sharing Playlists
* Better Searching
* Equalizing Volume of all songs
* Resizing Album Art


My main goal here though was just a Mixtape Player app without Ads that I can semi-easily add new mixtapes, make playlists, download songs for offline use, and use on all my devices.

I am happy that I accomplished those things. I think the scope creep starts happening later in the project when you learn how much you can do and what other apps have done. But you eventually lose enthusiasm for projects and are ready to move onto new ideas that excite you.

Web Version [here](https://mixtape.phantasmfour.com:8085/)

