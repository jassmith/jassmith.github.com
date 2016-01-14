---
layout: post
title: "Custom layouts with Xamarin.Forms, Part 1"
description: "Simple custom layouts"
category: Xamarin.Forms
tags: [Layout]
---
{% include JB/setup %}

Writing custom layouts in Xamarin.Forms gives you control over exactly where and how your controls appear on screen. Further in complicated layouts this can result in improved overall performance as the absolute minimum amount of work required to produce the layout is performed. This series assumes you are familiar with Xamarin.Forms. We will be covering the following topics:

- Simple custom Layout's
- Layout performance optimizations
- Animations inside of layouts
- Transitions between layout states

All layouts derive from the `Xamarin.Forms.Layout` class. This class provides the required mechanisms for adding and removing children internally as well as some key utilities for writing a layout. Further there is a generic subclass available, `Xamarin.Forms.Layout<T>` which provides a publicly exposed `IList<T> Children` that end-users can access. This collection can be typed to restrict the type of `View` the user can add, however most layouts should simply use `Xamarin.Forms.Layout<View>`.

We start by simply creating our class

{% highlight C# %}
public class CustomLayout : Layout<View>
{
	public CustomLayout ()
	{

	}
}
{% endhighlight %}

### LayoutChildren ###

The layout must to override the `LayoutChildren` method. This method is responsible for positioning children on screen. For this example a very simple algorithm producing a horizontal stack will be used. We will assume we always want to vertically fill our children.

{% highlight C# %}
// This method has some errors, do not copy!
protected override void LayoutChildren (double x, double y, double width, double height)
{
	for (int i = 0; i < Children.Count; i++) {
		var child = (View) Children[i];
		// skip invisible children

		if(!child.IsVisible)
			continue;
		child.Layout (new Rectangle (x, y, 100, 100));
		x += 100;
	}
}
{% endhighlight %}

Directly calling into `child.Layout` does not respect `child.VerticalOptions` or `child.HorizontalOptions`, so instead of using `child.Layout` it is prefered to use a call to `Layout.LayoutChildIntoBoundingRegion`. Further our layout does not currently attempt to measure the children at all to figure out the size they wish to be, instead everything is just hardcoded to 100x100. Children can be measured by using the `GetSizeRequest` method, which will return both the desired size and the minimum size the child desires. The updated method looks like:

{% highlight C# %}
protected override void LayoutChildren (double x, double y, double width, double height)
{
	for (int i = 0; i < Children.Count; i++) {
		var child = (View) Children[i];
		// skip invisible children

		if(!child.IsVisible) 
			continue;
		var childSizeRequest = child.GetSizeRequest (double.PositiveInfinity, height);
		var childWidth = childSizeRequest.Request.Width;
		LayoutChildIntoBoundingRegion (child, new Rectangle (x, y, childWidth, height));
		x += childWidth;
	}
}
{% endhighlight %}

This method will automatically be called whenever the layout needs to be recomputed. If your layout consists of hardcoded or fixed size elements, hard code their sizes into this algorithm instead of measuring. `GetSizeRequest` calls are some of the most expensive calls that can be made, and are not predictable in their runtime as the subtree may be arbitrary complex. Fixing their size is a great way to get a performance boost if dynamic sizing is not required.

### OnSizeRequest ###

Implementing `OnSizeRequest` is required to make sure the new layout is sized correctly when placed inside other layouts. During layout cycles this method may be called many times depending on the layout above it and how many layout exceptions are required to resolve the current layout hierarchy. It is therefor important to consider speed when implementing this function. Failure to implement this function will not always break your layout, particular if its always inside of parents which fix the child size anyway. However in most situations this is not the case.

{% highlight C# %}
protected override SizeRequest OnSizeRequest (double widthConstraint, double heightConstraint)
{
	var height = 0;
	var minHeight = 0;
	var width = 0;
	var minWidth = 0;

	for (int i = 0; i < Children.Count; i++) {
		var child = (View) Children[i];
		// skip invisible children

		if(!child.IsVisible) 
			continue;
		var childSizeRequest = child.GetSizeRequest (double.PositiveInfinity, height);
		height = Math.Max (height, childSizeRequest.Minimum.Height);
		minHeight = Math.Max (minHeight, childSizeRequest.Minimum.Height);
		width += childSizeRequest.Request.Width;
		minWidth += childSizeRequest.Minimum.Width;
	}

	return new SizeRequest (new Size (width, height), new Size (minWidth, minHeight));
}
{% endhighlight %}

By computing both a request size and a minimum size the layout is able to take part in overflow negotiation. This will allow for things like label truncation as needed or scrollable regions to be compressed and made to scroll.

That's it! That's a basic Xamarin.Forms custom layout. Of course this example is very simple and intended to provide you with the necessary context for the rest of the series.