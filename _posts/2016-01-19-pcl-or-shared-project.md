---
layout: post
title: "PCL or Shared Project?"
subtitle: "Which is right for you?"
category: General
author: Jason Smith
tags: []
---
{% include JB/setup %}

Since the dawn of time man has been faced with one question. Should I use PCL's libraries for my Xamarin.Forms projects, or should I use a Shared project? I'm here to tell you the answer is PCL, it is the way, the truth, and the light. Friends don't let friends use shared projects.

Okay so thats a bit strong but in general if you don't know what you should do, go PCL. If you have a strong reason to use a shared project, sure, but otherwise go PCL, your lack of #ifdef and spaghetti code will thank me later. Among other things, PCL will ensure that code you write is going to be portable not just to all current platforms, but any future platforms we might support as well.

Also I want to make sure everyone knows PCL is pronounced Pickle. Thats all.