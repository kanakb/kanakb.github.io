---
layout: post
title: "On Privacy-Preserving Apps"
description: ""
category: "posts"
tags: ["code","mobile"]
---
{% include JB/setup %}

[Snapchat](http://snapchat.com) is ridiculously popular with young smartphone users despite only one major differentiator: ephemeral messaging. For better or worse, supporting automatic message self-destruction has given Snapchat a reputation of being a _privacy-preserving_ app.

Great for them, right? It would be, if they actually [cared about privacy](http://www.forbes.com/sites/jjcolao/2014/01/02/the-hackers-who-revealed-snapchats-security-flaws-received-one-response-from-the-company-four-months-later/). Snapchat got [embarrassed](http://gigaom.com/2014/01/09/snapchat-says-sorry-for-getting-hacked-updates-app-with-phone-number-opt-out/). Then it got [embarrassed again](http://stevenhickson.blogspot.com/2014/01/hacking-snapchats-people-verification.html).

Having that privacy-preserving label makes life tougher for a developer. Let me give some context about what I mean by that.

### Musubi and Dispatch

For most of my time at Stanford, I worked on a mobile platform called [Musubi](http://mobisocial.stanford.edu/musubi). Here's why Musubi was interesting:

* Messages were signed and encrypted end-to-end on mobile devices
* Because the messages were encrypted, there was no central server that could read them
* The "public key" in this system was simply an identity: no key exchange issues!
* It provided messaging as a generic API for apps to use as a lightweight transport layer

Cool, but why do I need my lolcats to be end-to-end encrypted? I don't, nor does anyone else. That was the biggest problem with Musubi.

Despite that, there are some people that absolutely can make use of an app that does end-to-end encryption based on identities: journalists, their sources, and citizen reporters. These are the people who put their personal safety at risk in order to disseminate information and potentially [inspire social change](http://content.time.com/time/person-of-the-year/2011/).

Thanks to a generous grant by the [Brown Institute](http://brown.stanford.edu), I was able to work with some very talented folks in Computer Science and Journalism to understand what it means to build a privacy-preserving app, [Dispatch](http://dispatchapp.wpengine.com), for the people who need it the most.

### What I Learned

I won't get into everything that happened during the year that I worked on Dispatch. Instead, I'll just list out what I took away from the experience.

#### Not everyone knows they need protection

If you send email using a cloud service provider, it can get subpoenaed without your knowledge or consent. You don't need to be in a conflict region to get in trouble for sending the wrong picture.

#### The bad guys already have tools

People who break the law will fight through kludgy user experiences to stay anonymous.

#### The good guys care about user experience

This is a huge challenge: nobody will use your app if it's not easy to use. In the worst case, people will fall back on pen and paper, which is unfortunate to say as someone who works on technical solutions to problems.

#### Inventing your own secure protocol is hard

The first piece of advice any cryptographer will give is to use existing cryptography schemes and popular libraries. But what if those libraries don't perform well? What if they make key exchange incredibly difficult? Those are the problems that we had to solve in Musubi. We used a well-known scheme called [Identity-Based Encryption](http://crypto.stanford.edu/ibe/), but had to invent a custom message format to make it perform well. This message format was quite complicated, and [getting it right requires iteration](http://dispatchapp.wpengine.com/cryptographyfeedback/).

#### Your threat model needs to be crystal clear

The most important thing you can do as a privacy-preserving app developer is to say exactly what it means to be privacy-preserving. Under what circumstances will the app promise protection? More importantly, which situations will it not attempt to handle?

#### Peer review matters more here than anywhere else

[Cryptocat](http://crypto.cat) learned this the hard way and now goes to great lengths to encourage detailed security reviews.

#### It takes a dedicated team

It's not enough to just design a bulletproof protocol. Putting a complicated system into production is extremely time-consuming and error-prone. For our team of two students, it was practically impossible to do. The result was that the most sensitive component of our system, the key issuer, never became a distributed system. [And that single server got hacked](http://dispatchapp.wpengine.com/weve-been-hacked/), forcing us to start from scratch.

#### Your app doesn't exist in a vacuum

Even if you build the perfect messaging app with end-to-end encryption, there are innumerable external factors. Here are just a few points to consider:

* You need to use an identity that you don't use anywhere else. Otherwise, it's pretty easy to [link it back to your real name](https://www.usenix.org/conference/nsdi12/technical-sessions/presentation/roesner)
* You need something like [Tor](http://torproject.org) to mask your IP
* You need to encrypt your phone's storage in case it gets confiscated
* Having a private chat app on your phone could already be incriminating enough
* You might not even be legally permitted to use encrypted messaging

#### I'm not a designer

Actually, I already knew that.

#### Working in a cross-disciplinary team is awesome

[Seriously](https://scontent-b-sjc.xx.fbcdn.net/hphotos-prn1/936303_10151386620411537_122645508_n.jpg), do it.

### Papers

We wrote a couple short papers on this topic, which might be of interest:

[http://www2013.wwwconference.org/companion/p849.pdf](http://www2013.wwwconference.org/companion/p849.pdf)
[http://conferences.sigcomm.org/sigcomm/2013/papers/sigcomm/p459.pdf](http://conferences.sigcomm.org/sigcomm/2013/papers/sigcomm/p459.pdf)