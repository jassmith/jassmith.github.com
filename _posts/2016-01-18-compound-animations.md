---
layout: post
title: "Compound Animations"
subtitle: "A bit beyond the basics"
category: Animation
author: Jason Smith
tags: [animations]
---
{% include JB/setup %}

Animations in Xamarin.Forms work in basically the same manner all animations systems do. At the core is a timekeeper, called the Ticker which gives callbacks with timestep values, those timesteps are fed into a Tweener which then calls whatever callbacks the animation has registered with tweened values, finally the callbacks update properties on the Element. The end result is properties are being updated at about 60hz.

On top of this system are implemented some basic animations as extension methods on Element: `FadeTo`, `TranslateTo`, `ScaleTo` and so on. These accomplish about 90% of any animation needed by an app, however sometimes you just need to do something special. A complex animations containing multiple stages.

### Enter Xamarin.Forms.Animation ###

The animation class provides the basis for all Xamarin.Forms animations, as well as the capacity to compound them together. It serves as both low level tweener and storyboard. A basic animation wont need to use storyboarding. (It is worth noting this animation can be done much more simply with RotateTo, this is here for demo purposes!)

{% highlight C# %}
button.Clicked += (sender, args) => {
    var animation = new Animation(callback: d => button.Rotation = d, 
                                  start:    button.Rotation, 
                                  end:      button.Rotation + 360, 
                                  easing:   Easing.SpringOut);
    animation.Commit (button, "Loop", length: 800);
};
{% endhighlight %}

This produces the following animation.

<div class="center">
<video class="center" width="300" height="500" preload="metadata" controls=""><source src="/vid/rotate.mp4" type="video/mp4; codecs=avc1.42E01E, mp4a.40.2&quot;" /></video>
</div>

The easing is what is providing the overshoot effect. This is nice, however if we want to add more complexity to the animation, we can do so by building a compound animation. Compound animations are constructed from multiple internal animations.

{% highlight C# %}
button.Clicked += (sender, args) => {
    // Dirty hack you probably shouldn't use
    var width = Application.Current.MainPage.Width;

    var storyboard = new Animation ();
    var rotation = new Animation (callback: d => button.Rotation = d, 
                                  start:    button.Rotation, 
                                  end:      button.Rotation + 360, 
                                  easing:   Easing.SpringOut);


    var exitRight = new Animation (callback: d => button.TranslationX = d,
                                   start:    0,
                                   end:      width,
                                   easing:   Easing.SpringIn);

    var enterLeft = new Animation (callback: d => button.TranslationX = d,
                                   start:    -width,
                                   end:      0,
                                   easing:   Easing.BounceOut);

    storyboard.Add (0, 1, rotation);
    storyboard.Add (0, 0.5, exitRight);
    storyboard.Add (0.5, 1, enterLeft);

    storyboard.Commit (button, "Loop", length: 1400);
};
{% endhighlight %}

Instead of creating a single animation, we create 4, an outer animation into which we pack 3 children. The first child, `rotation`, runs for the entire animation. `exitRight` runs for the first 50% of the animation as specified by the parameters 0, 0.5. The `enterLeft` animation runs for the final 50%. Unlike the rotation argument, here we always hardcoding the start value, so as the tweener ticks over from the previous animation to the next one, the button is getting warped from the right edge of the screen to the left. The final result looks like this!

<div class="center">
<video class="center" width="300" height="500" preload="metadata" controls=""><source src="/vid/compound-anim.mp4" type="video/mp4; codecs=&quot;avc1.42E01E, mp4a.40.2&quot;" /></video>
</div>

Due to the way the rotation animation is coded, the button.Rotation value is increasing by 360 degrees every click. Worse if the button were double activated the button would become permanently stuck at an off angle as the value would be started from somewhere mid animation. For this reason it is generally advisable to have either the start or the end parameter be a hardcoded value. This prevents any screwy final states from occurring, like we do with the other animations.

Using the technique seen here it is possible to create a library of pre-built animation components which can be used to piece together more complex animations.