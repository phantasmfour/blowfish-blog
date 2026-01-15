+++
title = "Learning a language with Voice Synthesis"
date = "2023-08-16T20:24:43"
slug = "learning-a-language-with-voice-synthesis"
draft = false
+++

I have tried a lot of language learning apps. I find most of them are hard to stick to and don't always keep me interested or engaged. I have tried podcasts in other languages but they are sometimes too hard to understand. So with this we take an intermediate step.

I had an idea to basically create a podcast from any text I have that I find informative and engaging. And have lines/paragraphs read to me in English and then another lanaguage. This way you have an idea about what is going to be said and can learn new words while just listening to the audio and the stories.

{{< github repo="phantasmfour/coquiTTSArticles" showThumbnail=false >}}

Coqui TTS
---------

[Coqui](https://coqui.ai/) was the easiest way I found when looking to create natural sounding Voice Synthesis. They make it very easy to get started with Python modules and examples. The hardest part is going through all the voices.  
Coqui also has some paid features like using some of their better voices and cloning your own voice. Cloning a voice off limited audio is still very young in my opinion and was not very convincing. Their paid voices let you put emotion into the speech which would be very good for ingame characters or something different. For what I am working with the standard voices work just fine.

You can run Coqui code like this


```
    tts = TTS("tts_models/eng/fairseq/vits") 
    tts.tts_to_file(text="Be careful what you wish for!",file_path="output.wav")
    play_wav_file("output.wav")
    os.remove("output.wav")

```

Coqui is great and I found it had some of the most realistic voices models and the most accurate. I am running this script on a Raspberry Pi 4 which I figured would be up to the task. However Coqui takes a lot more RAM and CPU Cycles then I expected.  
  
I am translating around 30mins of audio from the text. This takes around 2 hours to finish running with my CPUs almost maxed out on the Pi. 

The Process
-----------

The breakdown of how the script runs is 

* Scrape the text from the article website
* Send the text to the Coqui functions to Synthesize Speech
* Use the Google Translate API to convert the text into your desired language
* Combine the output files into one large file with english and other language outputs intertwined.
* Upload the file to Discord for easy listening

The scraping process just uses BS4 which is good at what it does and just needs to be adapted for whatever html you are scraping. I ended up using the html parser and get\_text in order to get the best looking output. After that I filtered out any rouge characters and any lines I could find to remove. The article I am scraping has some Advertisements within so its hard to decern. But any text I know will almost always be there and I don't want to hear is removed.

Coqui handles all the text synthesis on its own and its mostly abstracted from the user. You pick which models and voices you like and it creates the Wav file for you. This runs a bit slow even on a modern PC so I decided to thread it to speed up this process.

There is a Python [library](https://github.com/ssut/py-googletrans) that is able to use the Google Translate API to make free and unlimited translations. Its pretty well known but I had to fight to find the correct version that worked with the example code.

Combining the files was a bit difficult since they are wav files. Basically I cut up the article into paragraphs/sentences that I feed to Coqui. This then outputs many files in both languages. I then wanted to hear the English version and then the second language. I also wanted these to flow so I created a half second blank audio file so they don't start talking exactly after the last audio is done playing. I name the files with an index so we know the order of when they were created and then basically just run a loop to concatenate all the file names together. Working with raw wav files is hard to concatenate, but using the pydub library you can add audio files to a segment and then export it as an mp3.

Discord webhooks are the very useful and versatile. They are also easy to work with and you basically just send a post request with your data and it goes to your Discord channel. This is convenient for me to listen to on the go without self hosting.

Issues
------

With any project there are issues.

### File Size

My first issue I hit when still developing was Discord does not let you upload files over 25MBs. I wrote some quick and very hacky code to check if our audio file is >25MBs and if so split it in half. I don't anticipate the files being >50MBs so this is safe for now.  
I then just submit two different post requests to upload the file. This can probably be done way more dynamically but it works.  
You can also just upload files to your own site or hos them somewhere else.

### Time

Running this script on a good Pi 4 takes around 2 hours even maxing out the CPUs. I made use of threading to speed up this process to the 2 hour mark. This is still bad but I can live with it.  
I tried to get around this by using [Piper](https://github.com/rhasspy/piper) . Piper gave me runtimes of around 10 minutes since its built for the Pi and was outputting lower quality audio. However the Spanish synthesis was glitchy and for every 1/30 words and it would get stuck just uttering gibberish. I did not love it and could not get around the problem by using other models. I also confirmed the text translation was correct it was indeed the model having issues. You can try your luck with other languages as the switch to using it in my code is a simple function change.

Bad Synthesis Example:
{{< audio "/audio/badSynthesis.mp3" >}}


### Memory

When I originally needed to move the Piper it was because using Coqui on the Pi 4 caused the script to use all of my memory and the OS would kill it. I tried including the Garbage Collector collection function to run more frequently but I cannot find where the memory leak is. Or just bad use of memory. I don't think I am holding onto any of these files but I could be. Or it could be within the Coqui library as that is the section where all my memory gets tied up.  
I slapped a band-aid over this problem by increase my swapfile to 10GBs.   
Yes not a good solution but it works. Overall I could have increased it to around 4-5GBs and I would have been fine.

Conclusion
----------

Overall it was cool to step into the Voice Synthesis game since I wasn interested more about it. I also got to look into more with the Voice Cloning so that was cool as well.

The project was good and sadly what took the most time was the memory issues which I did not really like as with Python I generally am not handling memory directly and worrying about it.

