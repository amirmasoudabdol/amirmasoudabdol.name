---
title: "Apple Notes vs. Hyperlinks"
layout: post
date: 2022-7-11
category: blog
author: amirmasoudabdol
description: Apple Notes does not know how to process hyperlinks consistently. The results vary between methods, and platforms, and there is no way to correct the unwanted behavior
image: 
  path: /assets/posts/apple-notes-vs-hyperlinks-summary-large-image.png
  alt: Apple Notes does not know how to deal with hyperlinks
twitter-image: /assets/posts/apple-notes-vs-hyperlinks-summary-large-image.png
tags: iOS iPadOS macOS UI UX
hidden: false # don't count this post in blog pagination
---


Over the past years, [Apple Notes](https://apple.co/3qy7gqc) has transformed from a simple notepad to a full featured note-taking app. It provides a long list of useful features like Formatted Text, Support for Attachments/Photos/Videos, Folders, Smart Folders, Handwritten Notes, Quick Notes, Scribble, Secure Notes, Note Sharing, and Live Collaboration. In iOS 16, Apple Notes will have an improved Secure Notes integration, and the Quick Notes is making its way into the iPhone. Overall, Apple Notes is really starting to be a great app, and after using so many note-taking apps (Drafts, Agenda, GoodNotes, Notability, DEVONthink, Workflowy), I just prefer something simple, and well-integrated just to make simple notes. Apple Notes' cross-platform support is sublime, the Apple Pencil support and integration is unmatched. 

Unfortunately though, these days, like most Apple software you start to notice inconsistencies and annoying behaviors here and there, e.g., there is no easy way to export your notes from Apple Notes. Unlike Apple Photos where you can simply export or drag/drop a photo out of the app, Apple Notes does not let you do that. Your only option is to export your entire library as a PDF, or use a third-party app which you'll never know when it stops working.

> *Apple Notes does not know how to deal with hyperlinks*.

Besides the lack of export functionality, one aspect of the app that I am particularly annoyed with is how Apple Notes manages hyperlinks. Apple Notes does not know how to deal with hyperlinks, and its behavior varies across systems and between methods. Here, I have listed the most consistent interactions and behaviors that I have found, but even these may not result in same outcomes based on the page that you are creating link from, or the time of the day I suppose! 

## macOS

### Method 1

Copying (Cmd + C) a link from Safari’s address bar results in a formatted link with the hyperlink itself being the text, e.g., [https://support.apple.com/en-us/HT205773](https://support.apple.com/en-us/HT205773).

- You cannot neither change the text, nor the link afterward, at least not trivially. You can edit the text, but if you are not careful you might delete the link itself.


### Method 2

<picture>
	<source srcset="/assets/posts/apple-notes-vs-hyperlinks-macOS-add-link-cmd-k-dark.png" media="(prefers-color-scheme: dark)">
	<img src="/assets/posts/apple-notes-vs-hyperlinks-macOS-add-link-cmd-k-light.png"  width="50%" align="right" float="right"/>
</picture>

By using the “Cmd + K” command on a selected text, you can create your own text and link it to a destination as you can do within most apps.

- After creating a link, you cannot change the text anymore, at least not trivially. You can still change the link itself by using the "Cmd + K" command.


### Method 3

You can create a rich link object by using the special "Add a link" toolbar button in the Apple Notes app. In this case, Apple Notes tries to guess which page you want to create a link from, and upon the selection, it **appends** a rich link object to your note.

- You cannot interact much with this object. You can only tell it to use the small image instead of the large image, or vice verse.
- You cannot make a list of these objects.
- The "Add a link" option is *only* available via the toolbar, and it should not be confused by "Add link" shortcut which you can instantiate using "Cmd + K".
- **Notice** how Apple Notes picks up some random information from the Yahoo! Finance app, and does not give me the opportunity to see all the open tabs on Safari for instance.

<picture>
	<source srcset="/assets/posts/apple-notes-vs-hyperlinks-macOS-add-a-link-dark.png" media="(prefers-color-scheme: dark)">
	<img src="/assets/posts/apple-notes-vs-hyperlinks-macOS-add-a-link-light.png"/>
</picture>


## iOS and iPadOS

### Method 1

Copying a link from Safari's address bar results in a formatted link with the hyperlink itself being the text, e.g., [https://support.apple.com/en-us/HT205773](https://support.apple.com/en-us/HT205773).

- You cannot neither change the text, nor the link; at least not trivially. You can very carefully edit it though.
- You get the same behavior if you use "Cmd + C" shortcut when using an external keyboard.

### Method 2

<picture>
	<source srcset="/assets/posts/apple-notes-vs-hyperlinks-iOS-copy-dark.PNG" media="(prefers-color-scheme: dark)">
	<img src="/assets/posts/apple-notes-vs-hyperlinks-iOS-copy-light.PNG" width="50%" align="right" float="right"/>
</picture>

Copying a link using the "Copy" option of the Shared Menu, *most of the times*, results in a link with the page's header as the text, e.g., [Use Notes on your iPhone, iPad, and iPod touch](https://support.apple.com/en-us/HT205773).

- If you are copying a link to a list, this action will break the list and it creates a new line!

### Method 3

You can create a rich link object by using the Share Menu, and creating a new note, or **appending** the rich link object to an already existing notes.

- Again, you cannot interact with this object much, and all the macOS rules are applied here.

<picture>
	<source srcset="/assets/posts/apple-notes-vs-hyperlinks-iOS-share-to-notes-dark.PNG" media="(prefers-color-scheme: dark)">
	<img src="/assets/posts/apple-notes-vs-hyperlinks-iOS-share-to-notes-light.PNG"/>
</picture>


## From iOS to macOS, and vice versa

With Universal Control, it is possible to copy a link from iOS, and bring it to the macOS; however, 

- If you copy a link from the address bar of Safari on macOS and paste it into a note on iOS, you will get the first behavior of the above lists, i.e., a formatted link with the hyperlink itself being the text, e.g., [https://support.apple.com/en-us/HT205773](https://support.apple.com/en-us/HT205773).
- If you copy the link using the "Copy" option from the Share Menu on iOS/iPadOS, you again you get the first behavior and not the link with the title.

## Possible Solution

I can imagine that they wanted to keep the Notes app as simple as possible, and they didn't want to add extra unnecessary features to it; however, here, it feels like that the hyperlink interactions' have been mostly overlooked. I say it is not about simplicity because Apple Notes has a proper table creation tools which arguably is quite complicated, certainly more than providing a box to rename a text/link.

<picture>
	<source srcset="/assets/posts/apple-notes-vs-hyperlinks-iOS-macOS-formatting-panel-dark.png" media="(prefers-color-scheme: dark)">
	<img src="/assets/posts/apple-notes-vs-hyperlinks-iOS-macOS-formatting-panel-light.png" width="50%" align="right" float="right"/>
</picture>

On both macOS and iOS, Apple Notes comes with a text formatting option, and text formatting menu; however, there is no option for making or interacting with a link there. Long pressing a rich link object offers you an option to use a small or large image, but not an option to convert the item to a normal link. Similarly, long pressing a “normal” link does not offer you anything, e.g., converting to a rich link object, or pulling the title of the page, or editing the text!

Apple Notes' UI is well-suited to deal with these problems and it can offer a consistent behavior but at the moment it does not, and that is frustrating! Maybe we can just have what it has been there for years in Pages, or Mails app, and stop this madness, please!

![](/assets/posts/apple-notes-vs-hyperlinks-pages-solution.png)
