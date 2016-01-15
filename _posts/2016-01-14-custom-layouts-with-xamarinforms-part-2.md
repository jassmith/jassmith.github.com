---
layout: post
title: "Custom layouts with Xamarin.Forms, Part 2"
author: Jason Smith
subtitle: "Layout performance optimization"
header-img: img/xamarin-platform.jpg
category: Xamarin.Forms
tags: [Layout Performance]
---
{% include JB/setup %}

[Part 1]({% post_url 2016-01-13-creating-custom-layouts-with-xamarinforms %})

When it comes to Layout performance Xamarin.Forms contains a very young layout system. I will be the first to admit that it is not the most blazing fast layout system in the world, however I like to think it's gotten to the point now where it is in the ballpark of acceptable. We are continuing to iterate on it and improve it with every cycle, and there is a lot of internal work going on to make sure that happens.

I want to digress slightly into what makes Layouts slow. In general layout performance issues arise when unbounded measure invalidation occurs. There are two key phases to this invalidation, and stopping the propagation earlier in the cycle results in better performance.

### The Invalidation Phase ###

![Invalidation Propagation](/img/invalidationupphase.png){: .center-block}

During the invalidation phase the each child informs the parent that it's measured size might change on the next measure call. Essentially this call informs the parent that any caches of measure calls are no longer valid. There is opportunity at this point to short circuit the cycle if the parent knows that the child's size will not change regardless of measurement results. An example of this would look like:

{% highlight XML %}
<Grid x:Name="parent">
	<Grid.RowDefinitions>
		<RowDefintion Height="50" />
	</Grid.RowDefinitions>
	<Grid.ColumnDefinitions>
		<ColumnDefintion Width="50" />
	</Grid.ColumnDefinitions>
	<Label Text="I am now a fixed size" />
</Grid>
{% endhighlight %}

No matter what the label measured size comes out to be, the Grid will always size it to be 50x50dp, so there is no need to propagate the event further up the hierarchy. More complex examples include using Star columns/rows when the Grid has a fixed size, or using ContentView's with WidthRequest and HeightRequest set on the child. However if you are not careful when crafting a Xamarin.Forms app, it is possible to allow these events to propagate to the top of the hierarchy, which is seriously bad juju. I will be giving a talk on this very topic at [Xamarin Evolve 2016](https://evolve.xamarin.com/).

### The Layout Phase ###

![Layout Propagation](/img/invalidationdownphase.png){: .center-block}

During the Layout phase, all parents of children which have received an invalidation event will relayout their children. Invalidation is quite expensive as it can easily impact parts of the tree which were logically nowhere near the original invalidation point. Propagating will stop if and only if the a child is layed out to the same size it was before the cycle began. This will exclude that part of the subtree from the rest of the layout cycle.

Unfortunately this is the most expensive possible place to have optimization taking place, it is significantly faster to prevent propagation in the first place.

### Caching Measurement Results ###

The most important optimization to perform is the caching of measurement results. This prevents the layout from having to remeasure every time [`OnSizeRequest`](https://developer.xamarin.com/api/member/Xamarin.Forms.VisualElement.OnSizeRequest/p/System.Double/System.Double/) is called. Building on the result from last time:

{% highlight C# %}
readonly Dictionary<Size, SizeRequest> measureCache = new Dictionary<Size, SizeRequest> ();

protected override SizeRequest OnSizeRequest (double widthConstraint, double heightConstraint)
{
	// Check our cache for existing results
	SizeRequest cachedResult;
	var constraintSize = new Size (widthConstraint, heightConstraint);
	if (measureCache.TryGetValue (constraintSize, out cachedResult)) {
		return cachedResult;
	}

	var height = 0;
	var minHeight = 0;
	var width = 0;
	var minWidth = 0;

	for (int i = 0; i < Children.Count; i++) {
		var child = (View) Children[i];
		// skip invisible children

		if (!child.IsVisible) 
			continue;
		var childSizeRequest = child.GetSizeRequest (double.PositiveInfinity, height);
		height = Math.Max (height, childSizeRequest.Minimum.Height);
		minHeight = Math.Max (minHeight, childSizeRequest.Minimum.Height);
		width += childSizeRequest.Request.Width;
		minWidth += childSizeRequest.Minimum.Width;
	}

	// store our result in the cache for next time
	var result = SizeRequest (new Size (width, height), new Size (minWidth, minHeight));
	measureCache[constraintSize] = result;
	return result;
}
{% endhighlight %}

Cached results must be cleared whenever the measurement of the layout is invalidated by any means.

{% highlight C# %}
protected override void InvalidateMeasure ()
{
	measureCache.Clear ();
	base.InvalidateMeasure ();
}
{% endhighlight %}

Cached results prevent propagation of the measure portion of the layout phase (which is the expensive part) and can result in dramatic speedups, especially in heavily nested scenarios. All default Xamarin.Forms layouts that benefit from caching already perform caching, so you would only need to implement this to add it to your own layout.

### Future Improvements ###

Unfortunately when the original API for the layout system was designed some information that is useful for optimization was not passed into key methods. [`InvalidateMeasure`](https://developer.xamarin.com/api/member/Xamarin.Forms.VisualElement.InvalidateMeasure/) does not pass along the reason for the invalidation, and even more important [`OnChildMeasureInvalidated`](https://developer.xamarin.com/api/member/Xamarin.Forms.Layout.OnChildMeasureInvalidated()/) does not pass along which child was invalidated. This has been resolved in internal API's however exposing these publicly requires an API break. Therefor the intention is to fix this with 3.0.

In part 3 we'll talk about animations inside of layouts.