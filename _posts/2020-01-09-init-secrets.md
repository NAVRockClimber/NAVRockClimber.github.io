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

My last post on [Handling Secrets](2019-12-16-handle-secrets-in-bc.md) started quite a few discussions. Which is good. Very good. We need to start thinking about security. Like I already said we are not playing in our on-premise backyard anymore where an network administrator will keep us save and secure. We entered a world where we working with connected systems. For now the number is probably quite small and handleable. If you ask my for my opinion this number will rise. We are working towards a mesh of smaller and bigger distributed services. The majority of them will be publicly in the internet and secured the one way or the other. If you try to categorize these discussions there have been clearly three main subjects of discussion. First how we can initialize the secrets. This is the subject of this blog post. The other subjects have been the mechanisms for authenticating and how to manage service like Azure Function. I probably will do posts about them in the future as well. But for now let us stick with initializing the secrets.

## TL;DR

## Ready player one

One of the first answers to my tweet about my blog entry have been that there would be now way to ship an app without an secret somewhere inside the app. Well, actually no. That was what I meant with an onboarding process. But I get the point of the author. It is the quick way. And as we all know, the quick way bites us in the ass one day and we are joining the dark side. Well the other way is quite some work and so let us stay for a moment on the path to the dark side.
What he probably meant was that we somehow need to get an initial secret shipped due to we have no way of reaching the customer and tell him the secret when he downloads our extension from the app source. If we somehow could get notified by Microsoft this would be awesome. In my dreams Microsoft would provide us with the ability to register a webhook and everytime our app gets installed we would get notified with some kind of unique identifier and mail address for the customer. When the webhook is invoked we could create an key and send the customer an e-mail with his initial credentials.

## Requirements for a secret

Before we dive deeper into the dark path let us first define a few requirements towards secrets. Let'S make a list and split the list into must haves and optional stuff.

- easy changeable
- easy updateable
- easy revokeable
- unique per customer
- separate from code

Well, you hopefully agree that these 5 bullet points are mostly non-negotiable requirements.

## The dark path

Unfortunately Microsoft does not provide us with the possibility of a webhook, yet. So let us be soft on the last point of our list. My first idea was to put it in the installer codeunit. But interestingly this does not work. I suspect Microsoft has not access to the the customer tenant at this point. If somebody knows the real answer shoot me a message on twitter. The second best option for initializing the secret is somewhere in the setup.

## Are our Requirements sufficient?

No, actually not. We looked at them from our point of view as ISV and partners. But if we take the customers position we recognize that we have to adjust these requirements a little bit. Let us assume the customer have to pay per usage for our services. Now the person with access to the keys leaves the company. Maybe he even leaves to a competitor. The customer probably won't like the idea that he still is able to access out services and want to change the keys.

- easy changeable (for ISV and Customer)
- easy updateable (for ISV and Customer)
- easy revokeable (for ISV and Customer)
- unique per customer
- separate from code

If we know look at the requirement we recognize we have a lot more to do. We need a website or portal we the customer can log in and request a new key. 
