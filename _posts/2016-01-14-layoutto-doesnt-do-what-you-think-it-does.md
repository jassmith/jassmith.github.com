---
layout: post
title: "LayoutTo doesn't do what you think it does"
subtitle: "But you can still do what you want!"
category: Xamarin.Forms
header-img: img/xamarin-platform.jpg
tags: [Layout, Animation]
---
{% include JB/setup %}

Okay I need to cop up to something here. I screwed up `View.LayoutTo` hard. Unfortunately it is too late to fix for 2.0 without a big API break, so I am writing this page to explain what it does, what it is useful for, what you probably want instead, and how to get the effect you wanted from `LayoutTo`. I would not be surprised if we fix this in some fashion for 3.0.

First what it is meant for. LayoutTo is actually used cause a child to be laid out in a series of animated steps to a new location. This sounds useful but there are a couple important things to note.

- It was only intended to be used by `Xamarin.Forms.Layout` subclasses, not externally.
- It is stupidly exposed in the same way the other animation methods are.
- It will initially appear to do what you want, but anything that triggers a relayout will cause your views position to reset to where it started.

Essentially the parent of the View you are calling LayoutTo on will not be aware of the translation/resize that happened, and will simply overwrite it at the next layout cycle (like when you rotate the device). This is because LayoutTo is calling the same method Layouts call to position children.

### What it is useful for ###

LayoutTo is useful if you are a layout and you want to animate transitions between layout states that contain both size and position changes. None of the default layouts do this, but it is actually trivial to make them do so during the layout pass. This will be covered in the series I am writing on custom layouts in Xamarin.Forms. Though I suspect the LayoutTo method will eventually be moved to live somewhere less harmful.

### What you want instead ###

Generally speaking what most users are looking to do is just translate things around the screen. This can be achieved with TranslateTo. TranslateTo is a *post* layout mechanism, and is thus respected by all layouts. There is also a solution if you want to both translate and resize, but that involves an absolute layout and is a topic I will resolve for a future post.