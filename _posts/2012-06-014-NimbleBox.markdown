---
title: "NimbleBox"
layout: post
date: 2012-06-14
category: project
image: /assets/projects/nimblebox/NimbleBox_Evernote_Logo.png
headerImage: true
author: amabdol
projects: true
description: NimbleBox
hidden: true # don't count this post in blog pagination
---

In early ages of the Evernote, I was trying to store and organize everything on the Evernote. I remember being inspired by the [Getting Things Done](https://en.wikipedia.org/wiki/Getting_Things_Done) technique and I set up a very sophisticated GTD system. I've defined several notebooks and tags for everything, _Inbox, Later, Today_, etc. The Evernote and my GTD system was working reasonably in my favor, I have collected a few thousand notes and managed to organize and finish some projects while keeping my sanity! 

One of the problem with Evernote client at that time was adding stuff to it was not very convenient, especially when you have large number of notes then the client would have been sluggish and rather buggy to work with. The Mac version was faster and more well-behaved. It was also equipped with this small version of Evernote called _Evernote Mini_. The Evernote Mini was a small app living in the menubar and it can be summoned by a HotKey to quickly capture a note, screenshot, or an audio recording. This was perfect for capturing quite notes and just dump them into the _Inbox_ for later processing. 

![Evernote Mini]({{"/assets/projects/nimblebox/EvernoteMini.png"}})

I love the idea of Evernote Mini, and I've tried to convince Evernote team to implement something similar for the Windows client. Unfortunatly that was not the priority at the time and it is still not in the Windows version. So, I've decided to implement something similar for Windows.

I’ve started with the main idea, *a simple and minimal app that can be used to quickly capture notes and send them to the Evernote.* Back then I was doing some C# programming and has never touched a real web API, but I knew about UI Design and I had the core experience in mind. Also, I was inspired by the _iA Writer_ and I wanted to reproduce the same simplicity and experience. The video below shows my attempts to recreate Evernote Mini for the Windows. I called it **NimbleBox**.

<iframe src="https://player.vimeo.com/video/68486671" width="640" height="483" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

In term of features, I’ve added a few neat tricks:

- Markdown
- Support for Right-to-Left languages like Farsi, Arabic, and Hebrew.
- Inline shortcut for adding `#Tags`, sending a note to a `@Notebook` or adding a `^DueDate` to a note.
- Twitter support
- Etc.

I was and I am still very happy with the outcome. Unfortunately, at around one year after I released the project, Aug 2013, one of the API that I was using stopped working and I didn’t have time to reimplement it myself. So the project effectively dead after that.