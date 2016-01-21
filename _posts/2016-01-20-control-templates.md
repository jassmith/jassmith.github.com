---
layout: post
title: "Control Templates"
author: "Jason Smith"
category: General
tags: [control templates, customization]
---
{% include JB/setup %}

Xamarin.Forms 2.1.0 has introduced ControlTemplates. For the uninitiated or unaware, Control Templating is a mechanism that allows separation of the logical view hierarchy from the visual hierarchy. Another way to think of it is a template that produces the visual hierarchy for your control or page. The concept can be confusing at first, however it is extremely powerful once understood and is best learned by example.

### A simple control template ###

To make a Control Template first we need a view which can be templated. Xamarin.Forms currently provides ControlTemplate properties on the following types:

- ContentPage
- ContentView
- TemplatedPage
- TemplatedView

The differences between each of these views is unimportant at this early stage, so we will work with the simplest, a `TemplatedPage`.

{% highlight C# %}
    public partial class LoginPage : TemplatedPage
    {
        public LoginPage ()
        {
            InitializeComponent ();

            // because its a demo damnit
            BindingContext = new LoginPageViewModel ();
        }
    }
{% endhighlight %}

And we will go ahead and apply a Style which we are fetching from our application resource dictionary

{% highlight xml %}
<?xml version="1.0" encoding="utf-8" ?>
<TemplatedPage xmlns="http://xamarin.com/schemas/2014/forms"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="TemplatingDemo.Views.LoginPage"
             Title="{Binding Title}"
             Style="{StaticResource LoginPageStyle }">
  <!--Remove Alt from above to use the other Style. This could also be set by implicit style-->
</TemplatedPage>
{% endhighlight %}

`LoginPageStyle` is defined inside the App.xaml resource dictionary

{% highlight xml %}
<ControlTemplate x:Key="LoginTemplate">
  <StackLayout VerticalOptions="CenterAndExpand" Spacing="20" Padding="20">
    <Entry Text="{TemplateBinding Username}" Placeholder="Username" />
    <Entry Text="{TemplateBinding Password}" Placeholder="Password" />
    <Button Command="{TemplateBinding Command}" Text="Click Here To Log In" />
  </StackLayout>
</ControlTemplate>

<Style TargetType="views:LoginPage" x:Key="LoginPageStyle">
  <Style.Setters>
    <Setter Property="ControlTemplate" Value="{StaticResource LoginTemplate}" />
  </Style.Setters>
</Style>
{% endhighlight %}

We now have enough context to evaluate what is happening here. The TemplatedPage has a `ControlTemplate` property which is being set via a Style named `LoginPageStyle`. Inside of the ControlTemplate there are items using a new syntax called TemplateBinding.

{% highlight xml %}
<Entry Text="{TemplateBinding Username}" Placeholder="Username" />
{% endhighlight %}

It should look similar because it works almost identically to a Binding, however the Source of the Template binding is always defined to be the Templated Parent, which in this case is the LoginPage. So what is it binding to? Good question, because some code was left out. The LoginPage exposes bindable properties for the template to bind to.

{% highlight C# %}
    public partial class LoginPage : TemplatedPage
    {
        public LoginPage ()
        {
            // SNIP
        }

        public static readonly BindableProperty UsernameProperty =
            BindableProperty.Create ("Username", typeof (string), typeof (LoginPage), null, BindingMode.TwoWay);

        public string Username
        {
            get { return (string)GetValue (UsernameProperty); }
            set { SetValue (UsernameProperty, value); }
        }

        // Other properties defined here as well, snipped for brevity
    }
{% endhighlight %}

In the end we get a page that looks like this:

![Control Template Result](/img/simple-control-template.png){: .center-block}

In effect then the Views inside the ControlTemplate are bound to values on the LoginPage which are in turn bound to the ViewModel in the normal fashion you would normally bind something. This might seem a bit heavy for this example, and to be honest it is.

### Why go through all the trouble? ###

There are several scenarios where this technique enables you to do things that were previously quite difficult. Mostly it comes down theming and making things beautiful. With ControlTemplates pages containing standardized content, your app can be easily themed and re-themed at runtime to have to different looks and feels while still presenting the exact same information. Just apply a different style with a different ControlTemplate and it looks brand new!

It is also worth noting that while this example looked into TemplatedPage, TemplatedView/ContentView offer much more flexibility and composability because they are Views and not full size pages. This means it is much more likely to see them presenting similar/identical data but with different views.

### Why introduce TemplatedView/Page? ###

TemplatedPage serves as a base class, replacing ContentPage as the most basic Page. Unlike ContentPage, TemplatedPage does not have a Content property. Therefor you cannot directly pack content into it. This means the only way to get content inside a TemplatedPage is to set the ControlTemplate, otherwise it will show up blank. The same is not true of a ContentPage, where the Content property can be assigned to as well as setting a ControlTemplate.

Setting both is where things start to get really interesting. If the ControlTemplate were modified to look like:

{% highlight xml %}
<ControlTemplate x:Key="LoginTemplate">
  <StackLayout VerticalOptions="CenterAndExpand" Spacing="20" Padding="20">
    <Entry Text="{TemplateBinding Username}" Placeholder="Username" />
    <Entry Text="{TemplateBinding Password}" Placeholder="Password" />
    <Button Command="{TemplateBinding Command}" Text="Click Here To Log In" />
    <ContentPresenter />
  </StackLayout>
</ControlTemplate>
{% endhighlight %}

And instead apply it to a ContentPage, the `ContentPage.Content` would end up inside the ContentPresenter in the ControlTemplate. The ControlTemplate effectively serves as an intermediate layer for the ContentPage and its Content.

There are lots of other neat things that can be done with ControlTemplates, and I'm sure many others will beat me to pointing them out, but I will try to hit some of the major ones as I find time.
