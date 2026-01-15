+++
title = "Bitwarden API Exploration"
date = "2023-03-11T18:58:38"
slug = "bitwarden-api-exploration"
draft = false
+++

I write a good amount of code and was looking for a way to store credentials. I already self host a Bitwarden instance and searches online show that even self hosted instances can use the Bitwarden API.

Bitwarden actually has two different APIs that I found. The [Organization API](https://bitwarden.com/help/api/) which is geared more to managing access to collections and passwords. From what I found you cannot use the organization API to retrieve any credentials. For this you need the [Vault API](https://bitwarden.com/help/vault-management-api/)  which lets you pull down all your credentials decrypted.

Accessing the Bitwarden Vault API
---------------------------------

The Bitwarden Vault API documentation points you to using the [Bitwarden CLI](https://bitwarden.com/help/cli/) to access the API. This is a CLI binary that you point at your Bitwarden instance and can then retrieve your credentials. It is very easy to install.   
Running `bw config server https://bitwarden` for me sent the bitwarden CLI to my local bitwarden instance. However I use a reverse proxy to handle all my TLS certs so I do not have to apply them everywhere. However the Bitwarden CLI is written in NodeJS and a self signed cert will throw errors. I attempted to point node to the cert using the environment variable `NODE_EXTRA_CA_CERTS=/path/to/cert.pem` but for some strange permissions error I could never get it to load. You can bypass cert inspection but just using the environment variable of `NODE_TLS_REJECT_UNAUTHORIZED='0'` I ended up doing this because I just wanted to explore the Bitwarden CLI to see if it could provide me with what I needed. I would not recommend this as bitwarden does have information on how to easily install a lets encrypt cert.

Is the Bitwarden API What I am looking for?
-------------------------------------------

TLDR: No. I am looking for an API that I could just provide an API Key to and pull a credential.

There are multiple ways to login to Bitwarden via the cli.  
Option 1: Provide Email and Master Password  
Option 2: API ID and Secret along with Master Password(?)  
Option 3: SSO

I don't have SSO setup in my homelab but I am pretty sure you still need to provide the master password to unlock you vault.  
My Bitwarden Collection contains all of my credentials since I use it for personal use. So needing to submit a master password is a no go for me. This was my biggest gripe with the API Key setup.   
Bitwarden offers you an API Client ID and Secret but these are used almost exactly as your email would if you went with option 1. In order to decrypt secrets you need to present your master key. 

When I was originally testing the Bitwarden CLI I had authenticated with my Email and Master password. You are then given a session that basically keeps you authenticated. I went and tried the API Client authentication next and still had the session open. This confused me as I thought using the API Client ID and Secret you would be able to access your credentials in the Vault. This is incorrect, but I thought I could.

I wanted to know how the API Client ID and Secret were able to do this so I setup [mitmproxy](https://mitmproxy.org/) and exported environmental variables in my terminal to get a look at how bitwarden was making the authentication request.   
Here is the total authentication flow using the API Client ID and Secret

![](/images/bw2nd.png)Lets step into the Post Requests

!["First Post"](/images/bw3rd_1st.png)Here we send a post to bitwarden with a payload containing the API Client ID and Client secret. We tell bitwarden we want to use the API. This looks similar to what the email authentication looks like except there they just send the email.

We then get back

![](/images/bw4th_1st.png)This is the response to the post request. You can see we are given back what I believe is the kdf algorithm you are using along with the kdf iterations. The [recommended iterations](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html#pbkdf2) vary for which algorithm you use. Interestingly enough there was a [bug](https://github.com/bitwarden/server/issues/589) in Bitwarden that forced iterations lower than 5,000 or above 2,000,000 before.  
I am unsure what the two Keys are for that you get back, I assume these are for decrypting the encrypted credentials that you get back. Lastly you get back an access token to just provide when you submit further requests.  
The next post request is a duplicated. I cannot figure out why that happens but it does...  
We can take a look at the sync get request

![](/images/bwAPIGet5th_1st.png)Here you get a response of what appears to be all of your credentials. They are all encrypted so you cannot actually derive anything from them without using your master password to decrypt them. The IDs are not encrypted here but you cannot tell what credential they map to without decrypting the name. This is important because you can pull a credential using an ID via the API.

  
At this point you are "logged in" and Bitwarden will tell you.  
But they will tell you in order to unlock(decrypt really) your vault you need to provide you master password.

![](/images/bwlogin6th_1st.png)At this point that all became useless to me as I would still have to provide the master password either way to decrypt the keys. Meaning I would always have to store that password somewhere. So why not just store the password I would need in code within a safe file that my script can read. 

When I started this I thought there was a way to read credentials just with the API Client ID and Secret but they just replace the username. This seems strange and I would love a way to have an API Token that would grant me access. Even some sort of 1 week validation on the token would be great so we don't have to rotate them manually if they are exposed.   
It seems like [other people](https://www.reddit.com/r/Bitwarden/comments/m3qo9v/cli_and_automation/) have run into this same issue and it seems like this may have been the intention with Bitwarden.  


Final Thoughts
--------------

Right now I think I will continue using files that contain my credentials sadly until I find a better way to get my credntials into code. Hopefully Bitwarden creates something like this in the future.

There are already python libraries which will let you interact with the Bitwarden CLI given you provide the master password.<https://pypi.org/project/ta-bitwarden-cli/#description>  
However I was looking for a solution without using the master password.  
I could have wrote a similar module with popen interacting with the Bitwarden CLI but this would not have helped me.

The only other option I saw for this was to create a user and add them to my Bitwarden organization and share a single credential I am using with them. This way I could hardcode a master credential somewhere and just expose one password to that account limiting the attack surface. Given the attack would need to first have someone on my network just for them to use those credentials to only have access to a handful of selected passwords this is Ok. But the more I thought about it this is really similar to just hardcoding the password I need somewhere in a file. The only advantage you get is dynamically selecting password to give that user access to.

![](/images/bwSteps7th_1st.jpg)

---

An interesting thing I found with Bitwarden CLI is they will let you setup a rest api server via the CLI. Then you can submit requests via the API. They make it seem like without this you won't have access to the API. I got this setup but I found that I would need to first be running it on my Bitwarden server and then start up the Bitwarden CLI and authenticate to it which would again require a hardcode of my master password.   
I never tried authenticating the CLI and then using the rest API in a hopefully authenticated manner but this would allow almost everyone on that network credentialed api access which does not sound good.  
But the interesting thing I found was that self hosted Bitwarden already responds to API requests like you saw in the mitmproxy flow. I think self hosted instances do this but maybe not the ones in the cloud which the `bw serve` command might be for to run an API from your host.

Overall interesting project and I got to learn a bit about a password manager I use.

