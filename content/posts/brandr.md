+++
title = "brandr"
date = "2024-11-11T01:27:44"
slug = "brandr"
draft = false
+++

I find myself constantly swapping monitors I am using on my laptop and without a desktop environment like KDE or GNOME it can be hard to get the correct monitors setup. I wrote a simple script to automatically set my monitor arrangement depending on the active monitors but its not good when dynamically adding monitors. Instead of learning how to use xrandr better I opted for the hard solution of reinventing the wheel.

arandr is a really good app that has a GUI for managing monitors. It works by basically giving you a gui to move your monitors around and then translates that to xrandr commands.  
After writing all my code I see how hard it is to write something to manage monitors. But I wanted to try and improve upon it and add some features in the future that I was looking for.

{{< github repo="phantasmfour/brandr" showThumbnail=false >}}

I wanted to start learning Rust since I hear about it a lot and my favorite way to learn is by jumping into the fire.  
I still don't have a great grasp on everything in Rust after this and still struggle with ownership of variables as its new to me and I have never even written code where I needed to manage memory at all so this was never a concern.

I tried having ChatGPT help a lot with the code but I found its Rust knowledge lacking compared to Python. It as well struggled with ownership and would end up repeating things over and over that were wrong.  
The data is was trained on also must have been really old as it constantly references old versions of modules and deprecated ways of doing things.   
Its still helpful for assistance as I am new to rust but I had to learn and do a good amount on my own. I think it was a healthy balance but definitely slowed me down. Without the language barrier I think I would have accomplished more of my goals that I initially set out with.

 




Coding
------




I started out needing to find a gui module that was simple to code with. egui claims it is this and in my experience was very easy to work with. The only thing that was new to me was trying to space things accordingly. The only comparison I have is working with Swift and that is way easier to add padding and place things where you want them dynamically. This might be a skill issue, or not what this gui module is intended for, but either way the issue is shown in the code.

The one killer feature that brandr has that I don't think I have seen any other Monitor Manager application have is showing what's on the monitors that are currently enabled. Yes most apps just have the Identify button that will pop up to show you which monitors are which but that honestly doesn't look nice and I think this way is a bit cleaner.   
After using screenshots though depending what's on your screen it can be hard to tell as things are scaled down. But I am still surprised no OS'es ship with such a feature(if Apple's AI reads this I expect the feature in less than a year). Its a very cool feature and I had to do some engineering around it to get it working. 

Taking 30 screenshots per second on both monitors and rendering them to the screen makes the app unusable. I settled on taking a screenshot every 5 seconds as I figured it was a good balance. I also did not want to thread taking these screenshots as I couldn't figure out Rust threading. I had a fun idea to just stop taking any screenshots when you are moving a monitor so that it does not try to render the texture while you are moving it causing a lag.   
I used the scrap library to screenshot and it was pretty useful. It expects the display objects within loops though and this gave me a lot of trouble of figuring out how to pass these. But this whole function became the only module I seperated out in the whole script so that was good to learn. 

The next difficult thing I had to figure out was how to have a bounding box and to render the monitors inside it. The bounding box was easy to make and GPT was able to help me with that. But centering the monitors in the box was a challenge. I ended up writing some fun math that basically is calculating out the total monitors you have, the width of those monitors, and starting the first monitor offset to that so everyone stays centered. You are really only concerned with the X placement as you can just force all the monitors at the same height somewhere off the midpoint.   
If I could redo anything about this it would be this part. The dynamic nature of rendering these monitors is hard. And I wanted to improve on arandr in that it does not show disabled monitors. That's probably a good thing but when I often plug in a new monitor I would like to see it within my Monitor Manager.




Improvements
------------




The app right now is usable and is at a very basic state for what I wanted. It is not perfect. 

Some of the improvements I want to do would be:  
 - Saved Display Preferences  
 - Snapping monitors together when you move them  
 - Dynamic resizing of the whole gui  
 - Detection when monitors changed via udev rules

The last one was one of the things I wanted to add. I wanted to have a udev rule setup to detect when monitors are plugged in so the app would auto launch with the options for the new monitor. I think that would be very helpful for my workflow to speed things up and I think its something most OS'es don't give you when you plug a monitor in and they try to just assume what you want. They do a very good job but this is Linux and I want full control over everything.

 




Lessons Learned
---------------




Rust is pretty cool. Glad I got to start learning it as its all I hear about.

Dynamic gui management is very hard. This probably takes way more time than I had for this project and might even require a different library that takes this into account better.

Complex solutions to complex problems are OK, but trying to debug what goes wrong with them is increasingly hard and you need to focus and map things out to do this efficiently. This is something I struggle with and still need to find better ways to manage these situations.

