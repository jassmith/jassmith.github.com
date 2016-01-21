---
layout: post
title: "Using Effects"
subtitle: "I didn't check the grammar"
author: Jason Smith
category: General
tags: [customization, effects]
---
{% include JB/setup %}

With the release of [Xamarin.Forms 2.1.0-pre1](https://www.nuget.org/packages/Xamarin.Forms/2.1.0.6500-pre1) we have introduced the new Effects API. Effects are methods for applying runtime changes to renderers for Views. This post will focus on advanced applications of effects for complicated use cases. Along the way I will point out areas where things could be drastically simplified for non-reusable scenarios.

However, I want to stress this, Effects are designed to be highly reusable. If an effect solves a problem, it can likely be applied to your app. If an Effect you wrote solved a problem for you, you could likely share that with others having the same problem. However if sharing Effects is going to be made easy, there needs to be a method to the madness. This post is attempting to show one approach to solving the madness.

{% highlight C# %}
public class ShadowEffect : PlatformEffect
{
	protected override void OnAttached ()
	{
		UpdateSize ();
		UpdateColor ();
		UpdateOpacity ();
	}

	protected override void OnDetached ()
	{
		Container.Layer.ShadowOpacity = 0;
	}

	protected override void OnElementPropertyChanged (PropertyChangedEventArgs e)
	{
		Debug.WriteLine (e.PropertyName);
		if (e.PropertyName == ViewExtensions.HasShadowProperty.PropertyName) {
			UpdateOpacity ();
		} else if (e.PropertyName == ViewExtensions.ShadowColorProperty.PropertyName) {
			UpdateColor ();
		} else if (e.PropertyName == ViewExtensions.ShadowSizeProperty.PropertyName) {
			UpdateSize ();
		}
	}

	private void UpdateOpacity ()
	{
		Container.Layer.ShadowOpacity = ViewExtensions.GetHasShadow (Element) ? 1 : 0;
	}

	private void UpdateColor ()
	{
		var color = ViewExtensions.GetShadowColor (Element);
		Container.Layer.ShadowColor = color.ToCGColor ();
	}

	private void UpdateSize ()
	{
		Container.Layer.ShadowRadius = (nfloat)ViewExtensions.GetShadowSize (Element);
	}
}
{% endhighlight %}

Whew, thats a monster! Just so we're clear it could have looked like this:

{% highlight C# %}
public class ShadowEffect : PlatformEffect
{
	protected override void OnAttached ()
	{
		Container.Layer.ShadowOpacity = 1;
		Container.Layer.ShadowColor = UIColor.Black.ToCGColor;
		Container.Layer.ShadowRadius = 6;
	}

	protected override void OnDetached ()
	{
		Container.Layer.ShadowOpacity = 0;
	}
}
{% endhighlight %}

Writing the effect this way is MUCH simpler, but also removes any chance of making it configurable and greatly reduces the re-usability for more than just the obvious reasons of no longer being parameterized. This is approximately what most of the older CustomRenderer approaches looked like when we were looking into this issue.

The ShadowEffect is created subclassing from PlatformEffect in the platform specific codebase. Like custom renderers, the implementations of PlatformEffects lives in the platform specific codebase, however the API for a Effect is identical across platforms, just deriving from `PlatformEffect<T, T>` with different generic parameters. One major difference of note is that Effects do not have type information about the Container/Control/Element they are attached to, this is because they can be attached to any Element. An effect needs to be able to either gracefully degrade or throw an exception when attached to an Element it doesn't support.

There are two important attributes to be set on any library containing an Effect.

- `[assembly: FormsResolutionGroupName ("YourCompany")]` : This is used to set a company wide namespace for your effects and prevents collisions with other effects with the same name. You can use the same group name in multiple assemblies.
- `[assembly: ExportEffect (typeof (ShadowEffect), "ShadowEffect")]` : This exports the effect with a unique ID which we will use along with the group name to locate the effect.

## Simple usage ##

Adding an effect to view at runtime is easy:

{% highlight C# %}
var button = new Button { Text = "I have a shadow" };
button.Effects.Add (Effect.Resolve ("YourCompany.ShadowEffect"));
{% endhighlight %}

If you don't export an effect for a particular platform, Effect.Resolve will return a non-null value which does effectively nothing. This makes handling cross-platform fixes that are unique to individual platforms much easier, but comes with a tiny memory penalty.

This is the point where we get off the boring train and get onto the hype train with a side of this-is-just-my-opinion soup. There is a much better way to do things with effects, which I hinted at above.

## All aboard the awesome train ##

{% highlight C# %}
public static class ViewEffects
{
	public static readonly BindableProperty HasShadowProperty =
		BindableProperty.CreateAttached ("HasShadow", typeof (bool), typeof (ViewEffects), false, propertyChanged: OnHasShadowChanged);

	private static void OnHasShadowChanged (BindableObject bindable, object oldValue, object newValue)
	{
		var view = bindable as View;
		if (view == null)
			return;

		var hasShadow = (bool)newValue;
		if (hasShadow) {
			view.Effects.Add (new ShadowEffect ());
		} else {
			var toRemove = view.Effects.FirstOrDefault (e => e is ShadowEffect);
			if (toRemove != null)
				view.Effects.Remove (toRemove);
		}
	}

	public static readonly BindableProperty ShadowSizeProperty =
		BindableProperty.CreateAttached ("ShadowSize", typeof (double), typeof (ViewEffects), 0d);

	public static readonly BindableProperty ShadowColorProperty =
		BindableProperty.CreateAttached ("ShadowColor", typeof (Color), typeof (ViewEffects), Color.Default);

	public static void SetHasShadow (BindableObject view, bool hasShadow)
	{
		view.SetValue (HasShadowProperty, hasShadow);
	}

	public static bool GetHasShadow (BindableObject view)
	{
		return (bool)view.GetValue (HasShadowProperty);
	}

	public static void SetShadowSize (BindableObject view, double size)
	{
		view.SetValue (ShadowSizeProperty, size);
	}

	public static double GetShadowSize (BindableObject view)
	{
		return (double)view.GetValue (ShadowSizeProperty);
	}

	public static void SetShadowColor (BindableObject view, Color color)
	{
		view.SetValue (ShadowColorProperty, color);
	}

	public static Color GetShadowColor (BindableObject view)
	{
		return (Color)view.GetValue (ShadowColorProperty);
	}

	class ShadowEffect : RoutingEffect
	{
		public ShadowEffect () : base ("Xamarin.ShadowEffect")
		{
			
		}
	}
}
{% endhighlight %}

Okay this looks like a lot of code, and it kind of is, but really we are just looking at 3 attached `BindableProperty`s. Nothing scary really, some static getters and setters. The only complex code is the `OnHasShadowChanged` which simply adds or removes the effect based on the value of the attached property. Lastly the code uses a RoutingEffect rather than directly calling Effect.Resolve just to make the removal process easier since there is no compile time access to the type information for the platform specific Effect.

Usage then looks like this:

{% highlight XML %}
<Button local:ViewEffects.HasShadow="True" 
        local:ViewEffects.ShadowColor="#222222" 
        local:ViewEffects.ShadowSize="4" />
{% endhighlight %}

or even better, use it in Style that you can apply to any/all Buttons:

{% highlight XML %}
<Style TargetType="Button">
  <Style.Setters>
    <Setter Property="local:ViewExtensions.HasShadow" Value="True" />
    <Setter Property="local:ViewExtensions.ShadowColor" Value="#232343" />
    <Setter Property="local:ViewExtensions.ShadowSize" Value="5" />
  </Style.Setters>
</Style>
{% endhighlight %}

Now you're writing effects like a boss.