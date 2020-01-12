---
layout: post
title:  "Initializing secret in business central"
date:   2020-01-09
desc: "Initializing secret in business central"
keywords: "Business Central,Secrets, Isolated Storage, Azure Function"
categories: [HTML]
tags: [Business Central,Secrets]
icon: fa-key
---

My last post on [Handling Secrets](2019-12-16-handle-secrets-in-bc.md) started quite a few discussions. Which is good. Very good. We need to start thinking about security. Like I already said we are not playing in our on-premise backyard anymore where an administrator will keep us save and secure. We entered a world where we working with connected systems. For now the number is probably quite small and handleable but if you ask my for my opinion this number will rise. We are working towards a mesh of smaller and bigger distributed services. The majority of them will be publicly in the internet and secured the one way or the other. If you try to categorize these discussions there have been clearly three main subjects of discussion. First how we can initialize the secrets - This is the subject of this blog post -. The other subjects have been the mechanisms for authenticating and how to manage service like Azure Function. I probably will do posts about them in the future as well. But for now let us stick with initializing the secrets.

## TL;DR

## Ready player one

One of the first answers to my tweet about my blog entry have been that there would be now way to ship an app without an secret somewhere inside the app. Well, actually there is. That was what I meant with an onboarding process in my last post. But I get the point of the author. It is the quick way. But as we all know, the quick way bites us in the ass one day and we are joining the dark side. Well, I got to admit, the other way is quite a piece of work. So let us stay on the path to the dark side for a moment.
What he probably meant was that we somehow need to get an initial secret shipped due to we have no way of reaching the customer and tell him the secret when he downloads our extension from the app source. If we somehow could get notified by Microsoft this would be awesome. In my dreams Microsoft would provide us with the ability to register a webhook and everytime our app gets installed we would get notified with some kind of unique identifier and mail address for the customer. When the webhook is invoked we could create an key and send the customer an e-mail with his initial credentials.

## Requirements for a secret

Before we dive deeper into the dark path let us first define a few requirements towards secrets. Let's make a list and split the list into must haves and optional stuff.

- easy changeable
- easy updateable
- easy revokeable
- unique per customer
- separate from code

Well, you hopefully agree that these 5 bullet points are mostly non-negotiable requirements.

## The dark path

Unfortunately Microsoft does not provide us with the possibility of a webhook, yet. So let us be soft on the last point of our list. Due to this is configuration data the best fitting place would be the installer codeunit. I don't like the idea really because we are mixing code and configuration data again. But, if there is some place where you could put it it is there. So, I extended the sample from my last blog post by the last line. You find it [here on github](https://github.com/NAVRockClimber/AlSecrectsExample/blob/DirtyInit/SecretsHelper.codeunit.al){:target="_blank"}.

```pascal
local procedure CreateSetup()
var
    SecretDemoSetup: Record "Secret Demo Setup";
    SecretHelper: Codeunit "Secret Helper";
begin
    SecretDemoSetup.Init();
    SecretDemoSetup."Function URL" := FunctionUrlLbl;
    SecretDemoSetup."Access Key Code" := 'AccessKey';
    SecretDemoSetup.Insert();
    SecretHelper.SaveSecret(SecretDemoSetup."Access Key Code", FunctionKeyTxt);
end;
```

Et voila, we initialized the secret during setup. Stored via an Helper codeunit in the isolated storage. Let's check it against our list. We still can change it, we can update it and we can revoke it. But, we cannot inform our customer about any of these changes. Unfortunately it is also not unique per customer and separate from code. This is what I mean with the need for an onboarding process. The simple fact of being able to change or revoke a key is simply not sufficient. We need a process around it.

## Theoretical solution

There we are. We initialized the secret but if we want to change it we still got a problem. We cannot change it because we cannot inform the anybody about it. But our problem is only half a technical problem. We are missing a process and we only thought about ourselves. We need to have our customers in mind as well. But if we take the customers position we recognize that we have to adjust the requirements a little bit. Let us assume the customer have to pay per usage for our services. Now the person with access to the keys leaves the company. Maybe he even leaves to a competitor. The customer probably won't like the idea that he still is able to access out services and want to change the keys.

- easy changeable (for ISV and Customer)
- easy updateable (for ISV and Customer)
- easy revokeable (for ISV and Customer)
- unique per customer
- separate from code

If we know look at the requirement we recognize we have a lot more to do. We need a website or portal where the customer can log in and request a new key.
But how do we get our customer on our website?

## The righteous path

Well, let us leave the dark side and return back onto the good side of the force. We still did not solve our initial problem in a satisfying way. I thought quite a while on how we can solve this issue. I solved it with the mighty force of JavaScript. Unfortunately JavaScript is not my strong suit. So, I beg you to show mercy with me if you find something ugly in it. And, yes it probably can look nicer but I am not really blessed with designer skills as well.

<img src="{{ site.img_path }}/InitSecrets/JSAddin.png" width="100%">

I like this variant much more. Let us present a nice text and a link you can follow by clicking. Maybe you can pimp it with your companies logo and more stuff that comes to your mind. But this is just a proof of concept.

## Force of JS

Ok let me show you the magic. The JavaScript in behind is actually quite simple. I defined an interface or a controladin in AL that is quite simple. The add-in get a text and a link and displays it.

```javascript 
controladdin "Url Forwarder AddIn"
{
    RequestedHeight = 300;
    RequestedWidth = 550;

    MinimumHeight = 200;
    MinimumWidth = 550;

    VerticalStretch = false;
    HorizontalStretch = false;

    VerticalShrink = true;
    HorizontalShrink = true;

    StartupScript = 'AddIn/Script/startupScript.js';
    Scripts = 'AddIn/Script/UrlForwarder.js';

    event AddInReady();

    procedure Show(Description: Text; UrlText: Text; Url: Text);
}
```

Also the JavaScript is quite simple. It consists in its core only of two functions. 

```javascript 
function Show(descriptionText, urlLabel, url)
{
    var r = CreateContent(urlLabel, descriptionText)
    r.onclick = function(e) {
        e.preventDefault();
        console.log(e);
        console.log(this);
        window.open(url, "_blank")
    }
}

function CreateContent(urlLabel, descriptionText)
{
    var paragraph = document.createElement("P"),
        paragraphText = document.createTextNode(descriptionText),
        hyperlink = document.createElement("a"),
        linkText = document.createTextNode(urlLabel);
    paragraph.appendChild(paragraphText);
    hyperlink.appendChild(linkText);
    hyperlink.href = '#';
    hyperlink.title = urlLabel;
    var addInElement = document.getElementById('controlAddIn');
    addInElement.appendChild(paragraph);

    return addInElement.appendChild(hyperlink)
}
```

The Show function is the one invoked from the page holding the add-in. Now we just create a paragraph section for the text and a hyperlink. At the the end we get the add-in element and attach both as children. All thats left than is to implement the onclick function and open the link in a new window.

Now we just need to show the add-in. For demo purposes I check if BC can retrieve a secret from my isolated storage and if not I present a page with the add-in.

```pascal
[NonDebuggable]

procedure GetSecret(KeyName: Text): Text
var
    SecretValue: Text;
    PlainBuffer: Text;
    EncryptionErr: Label 'The secret cannot be stored. The encryption is not enabled.';
begin
    if not EncryptionEnabled() then begin
        error(EncryptionErr);
    end;
    if not IsolatedStorage.Contains(KeyName, DataScope::Module) then begin
        InitSecret();
    end;
    if not IsolatedStorage.Get(KeyName, DataScope::Module, SecretValue) then begin
        exit('');
    end;
    PlainBuffer := Decrypt(SecretValue);
    exit(PlainBuffer);
end;
```

If the isolated storage does not contain a secret with the corresponding key we call the InitSecret function and the page is shown.

As always you can find the whole code for the example on [github](https://github.com/NAVRockClimber/ALInitSecrectsExample){:target="_blank"}.