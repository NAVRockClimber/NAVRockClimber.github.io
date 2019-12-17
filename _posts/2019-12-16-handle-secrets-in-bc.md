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

On my last visit at the NAV TechDays I had a nice chat with Wael AbuSeada the security expert from Microsoft and Gert Robyns (Principal Software Engineer at Microsoft) on handling secrets in Business Central. At almost the same time I got a mail forwarded that an access key should be changed. After digging a bit into the mail I found code similar to the snippet below.

```pascal
local procedure GetKey(): Text
begin
    exit('U7YkzUyaXhgQFBgMMWicSwIe7xcYD0aAPFwj3pjZNPs6YCrGpwLJIg==');
end;
```

I immediately had this feeling telling me this is wrong on so many levels. At least the function is local function and you do not blare it to every app installed with you. Unfortunately I had to learn later that many NAV/BC developers do not know what the problem is with situation this.

## **The TL;DR**

Secrets are metadata and don't belong into code and you should not spread them via mail. Secrets should be changeable, stored securely and never come into the near of code or a repository. One way of doing this I show in my [example repository](https://github.com/NAVRockClimber/AlSecrectsExample).

## **What's the problem?**

With the situation above we have several Problems.

1. Secrets in code are a huge potential security issue. Cardinal errors like this cause situation where we read in the press that thousands of personal records have been leaked. We should never be in the need to inform developers about a changed access key. An access key is configuration data and should as sure as hell not be distributed via e-mail. Our code and onboarding process should allow to change a key without the need of changing one line of code. 

2. If your key gets compromised you have to roll out an update. And, rolling out an update can take quite some time. Time you might not have in terms of revoking your compromised key. You should simply be able to revoke the key, generate a new one and inform your customer to visit your website and grab a new key. No need to say the website should be protected by some kind of login.

3. Why is this piece of code not in a central library that sleeps in a repository? With a repository we simply make a pull request for submitting a change. This change than gets distributed via a pipeline and in the best case your developers do not even recognize that something changed. It almost seems like the developers copied the same piece of code. Which is bad. Really bad. Ok, this is probably where we partially have to blame Microsoft. In a matter of fact it is simply not documented that you can publish something Microsoft calls a library app. Library apps are not shown in Microsoft's app source and you ship them with you "real" App. Unfortunately this comes with a caveat. If you got multiple apps referencing the library app all your apps have to reference the same version due to you cannot install one app twice. 

In the Business Central world everybody tells us nowadays we have to use Azure Functions now. But rarely we hear people speaking about security. We have to learn that we might not play in our on premise backyard anymore. We cannot rely anymore on an admin isolating us from the rest of the world. We communicate with services in the big world wide web. This is an area where we need to have an eye on security. At least have it in the back of our minds. 

## **What's a secret anyway**

A secret can be a password, a server configuration, token, certificate and more. A secret has certain properties like it should be able to be changed at every time. The good thing is nearly every language and tool in our modern world has a way of dealing with secrets. Docker has its secrets file, Azure Pipelines have private variables preventing them from being printed in a log, the BASH has HISTCONTROL and won't store commands in its history starting with a blank. ASP.Net and Visual Studio have mechanisms for managing secrets and Azure got its KeyVault. Secrets should be kept separate from your code and away from git. Because neither the internet nor git really forgets. A quick search on Github for "delete password" reveals 34 million commits.

## **Come on dude, get to the point**

In Business Central we got the **[Isolated storage](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-isolated-storage)** and this is where secrets belong. Don't get me wrong. It is neither the only way to store a secret nor is it rock solid secure. We should handle secrets like something that can always be comprised by accident and we should have a plan b for situations like that.

### How to store a secret

In an ideal word we could throw the secret into the [isolated storage](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-isolated-storage) and it would store it encrypted for us. Well, there is good news and bad news. The bad news first. Unfortunately Microsoft did in my opinion not a good job here. We can only store **encrypted** secrets whose length do not exceed 215 characters. In the real world like when you are dealing with OAuth tokens we exceed this character limit. The good news is access tokens for Azure Functions are shorter. Also we can work around this limitation with Microsoft's Encryption API. As I will show in the following snippet.

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

For storing the secret encrypted we first check if the encryption is [enabled](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/system/system-encryptionenabled-method) at all. After this we just throw it in the [encryption](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/system/system-encrypt-method) module and store our secret. Nice, fine and secure. But, pay attention here and set the NonDebuggable attribute. We do not want to expose our secret accidentally in the debugger.

### Retrieve a secret

For retrieving a secret we just need to reverse the procedure. We first fetch the encrypted secret from the **[isolated storage](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-isolated-storage)**, **[decrypt](https://docs.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/system/system-decrypt-method)** it and return the value. You find the full on my github repo [here](https://github.com/NAVRockClimber/AlSecrectsExample).

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

We can now write and retrieve secrets. For implementing the full CRUD functionality only the delete method is missing. But due to you are still with me and as long as you understood the examples above I assume you are capable of writing them on your own or just copy them from my [repo](https://github.com/NAVRockClimber/AlSecrectsExample). 

If you now decorate all this with a pipeline and come up with a onboarding and update process for your customers or consumers of your Azure Function you should be safe.