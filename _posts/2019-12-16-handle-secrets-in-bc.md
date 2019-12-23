---
layout: post
title:  "Handling secrets in business central"
date:   2019-12-23
desc: "Handling secrets in business central"
keywords: "Business Central,Secrets, Isolated Storage"
categories: [Secrets]
tags: [Business Central,Secrets, Isolated Storage]
icon: fa-key
---

On my last visit at the NAV TechDays I had a chat with Wael AbuSeada (Security Expert at Microsoft) and Gert Robyns (Principal Software Engineer at Microsoft) on handling secrets in Business Central. I tried to save an OAuth key in the Isolated Storage and wanted their opinion on that. A few days later back home I found a piece of code in repo, similar to the one below. It was used for authenticating against an Azure Function. After a bit of digging I had to learn there was even a mail send around with a key that should be changed. I immediately felt like tearing my hair out. 

```pascal
local procedure GetKey(): Text
begin
    exit('U7YkzUyaXhgQFBgMMWicSwIe7xcYD0aAPFwj3pjZNPs6YCrGpwLJIg==');
end;
```

A few chats later I was sitting totally frustrated in my office. I spoke to the developers of this piece of code, a few other developers and unfortunately I had to learn later that many NAV/BC developers do not know what the problem is with the code above. So, here are my 2 cents on why this is wrong and how you can make better. 

## **The TL;DR**

Secrets are metadata and don't belong into code and you should not spread them via mail. Secrets should be changeable, stored securely and never come into the near of code or a repository. One way of doing this I show in my [example repository](https://github.com/NAVRockClimber/AlSecrectsExample){:target="_blank"}.

## **What's the problem?**

Let me explain first why this is a problem and what's the situation with the code above. Actually there are several problems.

1. Secrets in code are a huge potential security issue. Cardinal errors like this cause situations where we read in the press that thousands of personal records have been leaked. We should never be in the need to inform developers about a changed access key. 

2. An access key is configuration data and should as sure as hell not be distributed via e-mail. We should be able to change a key without informing everybody. In an ideal world nobody in development should even recognize that the key was changed. Our code and onboarding process should allow to change a key without the need of changing one line of code. 

3. If your key gets compromised you have to roll out an update. And, rolling out an update can take quite some time. Time you might not have in terms of revoking your compromised key. You should simply be able to revoke the key, generate a new one and inform your customer to visit your website and grab a new key. No need to say the website should be protected by some kind of login.

4. The fact of one key is being send around shows the next problem. This means everybody is using the same key. Meaning revoking the key and creating a new key will affect all consumers of you azure function. In a case of a compromised key you got to contact and inform all your customers. If you even forced to revoke a key for a vital function you might stop all your customers from being able to work. In the worst case you might face financial claims due to damage compensation.

5. The fact everybody consuming this Azure Function has been informed via mail raises the question if there is no central repo or library. In a repository we simply make a pull request for submitting a change. With the next pull everybody should have the update. In a library a build pipeline will distribute the change. Informing people via mail of changes is really bad. Ok, this is probably where we partially have to blame Microsoft. In a matter of fact it is simply not documented that you can publish something Microsoft calls a Dependency App. Library apps or Dependency Apps are not shown in Microsoft's app source and you ship them with you "real" App. 

With the new modern development and Microsoft pushing towards the cloud everybody tells us nowadays we have to use Azure Functions. But rarely we hear people speaking about security. We have to learn that we might not play in our on premise backyard anymore. We cannot rely anymore on an admin isolating us from the rest of the world. We communicate with services in the big world wide web. This is an area where we need to have an eye on security. At least have it in the back of our minds. 

## **What's a secret anyway**

A secret can be a password, a server configuration, token, certificate and more. A secret has certain properties like it should be able to be changed at every time. The good thing is nearly every language and tool in our modern world has a way of dealing with secrets. Docker has its secrets file, Azure Pipelines have private variables preventing them from being printed in a log, the BASH has HISTCONTROL and won't store commands in its history starting with a blank. ASP.Net and Visual Studio have mechanisms for managing secrets and Azure got its KeyVault. Secrets should be kept separate from your code and away from git. Because neither the internet nor git really forgets. A quick search on Github for "delete password" reveals 34 million commits.

## **Come on dude, get to the point**

In Business Central we got the [Isolated Storage](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-isolated-storage){:target="_blank"} and this is where secrets belong. Don't get me wrong. It is neither the only way to store a secret nor is it rock solid secure. We should handle secrets like something that can always be comprised by accident and we should have a plan b for situations like that. Meaning or code should allow to change the secret.

### **How to store a secret**

In an ideal word we could throw the secret into the a function and it would store it encrypted for us. Wait, we got something like this. It is called [Isolated Storage](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-isolated-storage){:target="_blank"}. But well, there is good news and bad news. The bad news first. Unfortunately Microsoft did in my opinion not a good job here. We can only store **encrypted** secrets whose length do not exceed 215 characters. In the real world like when you are dealing with OAuth and tokens we might exceed this character limit. The good news is access tokens for Azure Functions are shorter. Also we can work around this limitation with Microsoft's Encryption API. As shown in the following snippet.

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

For storing the secret encrypted we first check if the encryption is [enabled](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/system/system-encryptionenabled-method){:target="_blank"} at all. After this we just throw it in the [encryption](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/system/system-encrypt-method){:target="_blank"} module and store our secret. Nice, fine and secure. But, pay attention here and set the NonDebuggable attribute. We do not want to expose our secret accidentally in the debugger.

### **Retrieve a secret**

For retrieving a secret we just need to reverse the process. We first fetch the encrypted secret from the [Isolated Storage](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-isolated-storage){:target="_blank"}, [decrypt](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/system/system-decrypt-method){:target="_blank"} it and return the value. You find the full example in my github [repo](https://github.com/NAVRockClimber/AlSecrectsExample){:target="_blank"}.

```pascal
[NonDebuggable]
procedure GetSecret(KeyName: Text): Text
var
    SecretValue: Text;
    PlainBuffer: Text;
begin
    if IsolatedStorage.Contains(KeyName, DataScope::Module) then begin
        if IsolatedStorage.Get(KeyName, DataScope::Module, SecretValue) then begin
            if EncryptionEnabled() then begin
                PlainBuffer := Decrypt(SecretValue);
                exit(PlainBuffer);
            end;
        end;
    end;
end;
```

We can now write and retrieve secrets. For implementing the full CRUD functionality only the delete method is missing. But due to you are still with me and as long as you understood the examples above I assume you are capable of writing them on your own or just copy them from my [repo](https://github.com/NAVRockClimber/AlSecrectsExample){:target="_blank"}. 

If you put all this in a central library, decorate this with a pipeline and come up with an proper onboarding and update process for your customers you should have a quite solid solution to dealing, distributing and updating your secrets.