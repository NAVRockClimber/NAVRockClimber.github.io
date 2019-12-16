---
layout: post
title:  "Handling secrets in business central"
date:   2019-12-16
desc: "Handling secrets in business central"
keywords: "Business Central,Secrets, Isolated Storage"
categories: [Secrets]
tags: [Business Central,Secrets, Isolated Storage]
icon: fa-key
---

On my last visit on the NAV TechDays I had a nice chat with Wael AbuSeada the security expert from Microsoft and later with Gert Robyns (Principal Software Engineer at Microsoft) on handling secrets in Business Central. At almost the same time I received an email forwarded that a secret should be changed from a function with a hard coded secret. Obviously the code was not in a central library. Instead every developer has copied it.

After digging a bit into the mail I found code similar to the snippet below. You know these situation where you immediate feel this is wrong on so many levels? 

```pascal
local procedure GetKey(): Text
begin
    exit('U7YkzUyaXhgQFBgMMWicSwIe7xcYD0aAPFwj3pjZNPs6YCrGpwLJIg==');
end;
```

At least it is a local function and you you do not blare it to every extension installed with you. Unfortunately I had do learn later that many NAV/BC developers do not know what the problem is with situation this.

## **The TL;DR**

## **Whats the problem?**

1. Why is this piece of code not in a central library sleeping in a repo and all developers have to be informed via email? Obviously all developers copied the same piece of code. Which is bad. Really bad. Ok, this is probably where we partially have to blame Microsoft. In a matter of fact it is simply not documented that you can publish - let us call them - library extension that will not be shown in Microsoft's store. Although this comes with a caveat. If you got multiple extension referencing the library your extensions have to reference the same version.

2. Secrets in code are huge potential security issue. Cardinal errors like this cause situation where we read in the press that thousands of personal and/or sensitive records have been leaked.

3. If your key gets compromised you have to roll out an update. With all the bad things starting from the beginning. And, rolling out an update in the cloud area can take a time. Time you might not have in terms of revoking your compromised key. 

In the Business Central world everybody tells us we have to use Azure Function now. But rarely we hear people speaking about security. We have to learn that we might not play in our on premise backyard anymore. We cannot rely anymore on an admin isolating us from the rest of the world.  We communicate with services in the big world wide web. This is an area where we need to have an eye on security. At least have it in the back of our minds. 

## **What's a secret anyway**

A secret can be a password, a server configuration, token or certificate. A secret has certain properties like it should be able to be changed at every time. The good thing is nearly every language and tools in our modern world has a way of dealing with secrets. Docker has its secrets file, Azure Pipelines have private variables preventing them from being printed in a log, the BASH has HISTCONTROL and won't store commands in its history starting with a blank. ASP.Net and Visual Studio have mechanisms for managing secrets and Azure got its KeyVault. Secrets should be kept separate from your code and away from git. Because neither the internet nor git really forgets. A quick search on github for "delete password" reveals 34 million commits.

## **Come on dude, get to the point**

In Business Central we got the **Isolated storage** and this is where secrets belong. Don't get me wrong. It is neither the only way to store a secret nor is it rock solid secure. It is for hiding it from the eye-catching and giving you a certain degree of security from other extensions. Also if a access keys is the best way to secure an azure function should be open for debate.

### How do I store a secret?

In an ideal word we could throw the secret into the isolated storage and it would store it encrypted for us. Well, there is good news and bad news. The bad news first. Unfortunately Microsoft did not a good job here in my opinion. We can only store **encrypted** secrets whose length do not exceed 215 characters. In the real world like when you are dealing with OAuth tokens we exceed this character limit. The good news is access tokens for Azure Functions are shorter. Also we can work around this limitation. As shown in the following snippet.

```pascal
[NonDebuggable]
procedure SaveSecret(KeyName: Text; SecretValue: Text)
var
    EncryptionErr: Label 'The secret cannot be stored. The encryption is not enabled.';
    EncrytedBuffer: Text;
begin
    if not EncryptionEnabled() then begin
        Error(EncryptionErr);
    end;
    EncrytedBuffer := Encrypt(SecretValue);
    IsolatedStorage.Set(KeyName, EncrytedBuffer, DataScope::Module);
end;
```

For storing the secret encrypted we first check if the encryption is enabled at all. After this we just throw it in the encryption module and store our secret. Nice, fine and secure. But, pay attention here and set the NonDebuggable attribute. We do not want to expose our secret in the debugger.


