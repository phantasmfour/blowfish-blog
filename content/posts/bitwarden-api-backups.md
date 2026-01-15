+++
title = "Bitwarden API Backups"
date = "2023-03-12T00:58:07"
slug = "bitwarden-api-backups"
draft = false
+++

From my [last post](/posts/bitwarden-api-exploration.md) I got to experiment with the Bitwarden API and learn the different levels of authentication. I had an interesting idea to leverage the API to create backups of my Bitwarden credentials. However there are no actual documented API commands to do this and every way via the GUI requires you provide the master password. But I was able to find a way.

How?
----

When you login via the API the first thing your client does is do a get request for /api/sync. This returns back all of your encrypted credentials and everything you would pretty much need. The thought is that everything is done on the client side so that the server does not ever get your master password. So you are given back JSON of encrypted passwords and you did not need to provide the master password to get this.  
  
My first attempts to decrypt this was to format similar to how bitwarden already exports its encrypted password files.  
This did not succeed however because the key that you are given when you export your credentials contains your master password somewhere within. When bitwarden gets it back it is able to decrypt it.  
I am unauthenticated so I do not have that key just two keys that are used by the client to decrypt the data. 

![](/images/bw2nd_2nd.png)So at this point I know that the clients get this data back and somehow they are able to decrypt this into password. Bitwarden is open source so it should not be that hard to find. Well after looking myself and reading [blogs](https://ildella.net/2019/11/18/bitwarden-part1-password/) from some smarter people this is harder than I thought.

Someone has probably done something similar
-------------------------------------------

I happened upon a script called [BitwardenDecrypt](https://github.com/GurpreetKang/BitwardenDecrypt) from Gurpreet. After reading the script it looked very similar into what I wanted to do. So instead of editing Gurpreet's code to do what I wanted I figured I would edit my script to have output parsable to his. BitwardenDecrypt was written around the same exact principle. When you login to bitwarden via the Desktop app you are given the same data back but in a different format. This file is saved to data.json. Luckliy I had one of these from my previous tests. I was able to take the format needed and create a json file that was structured for that. After that all you have to do is run BitwardenDecrypt and with your master password your data is decrypted.

This works great actually and is everything I wanted. No actual authentication needed to Bitwarden and I have a backup of my passwords incase my self hosted instance goes down. The way BitwardenDecrypt formats the data back into a JSON you can even take the output save it to json and restore your password in the bitwarden gui like you would with a regular unencrypted backup.

I currently have my script setup as a daily cron job to pull encrypted backups from Bitwarden. I uploaded the code to github for others to have a go at it.

{{< github repo="phantasmfour/bitwardenEncryptedBackup" showThumbnail=false >}}