---
layout: post
title: "Should you support landscape?"
date: 2015-12-21 15:49:18 -0500
comments: true
categories: Android
---
Android fragmentation is no joke. There are a total of [~24,000 unique Android devices and 10 different versions of the OS in use as of 2015](http://opensignal.com/reports/2015/08/android-fragmentation/). As Android developers, we want to support as many different devices and versions as possible but doing so can feel like going down a rabbit hole.

{% img http://chrisarriola.me/images/android_device_fragmentation_2015.png 750 500 %}

On top of device and OS fragmentation, having to supporting yet another screen variant, landscape, adds even more complexity.

To simplify things, wouldn't it be great to lock the orientation mode of all screens to portrait and just say “our app doesn't support landscape?” And hey, while we're at it, “our app doesn't support anything below the latest version of Android, too.”

Seriously though, should you support landscape on Android?

Like any product decision, *the answer isn’t always quite that simple*.

There are a set of applications and screens where landscape makes sense. Say for example, when playing games, or, when watching videos. However, an overwhelming majority of apps used on phones are in portrait.

Based on a [mobile UX survey done by UXmatters in 2013](http://www.uxmatters.com/mt/archives/2013/02/how-do-users-really-hold-mobile-devices.php), only 10% of users orient the phone in landscape when using their phone with two-hands. This does not surprise me. Think about it, when was the last time you used your phone and an unintended screen rotation occurred? It can be annoying.

Because of these reasons, I can see why it is tempting to lock an app to only support portrait mode.

I can think of one good reason though why you shouldn’t do this and consider supporting landscape: **configuration changes**.

Configuration changes occur in runtime and are caused by various events such as: when keyboard visibility changes, when language changes, or  when orientation changes. This in turn causes the current foreground Activity to be reconstructed once the change finishes and all instance state will be recovered using Android’s parcelling mechanism via Bundle.

Framework views automatically handle saving/restoring state (ethis is why you don't lose the content in an `EditText` on a configuration change); however, a custom view, an Activity, or a Fragment instance state will not be automatically recovered by the framework. Instead, this needs to be explicitly retained in certain life cycle events. It's really easy to miss an instance state that should be recovered, especially when orientation is locked, and so it's good practice to test configuration changes while developing. *Changing the orientation of the device is the simplest way to simulate that*.

Here's an example of how you should retain state across configuration changes in an `Activity`:

```
public class MainActivity extends Activity {

    private int someIntValue;

    @Override
    public void onSaveInstanceState(Bundle savedInstanceState) {
        // This is called by the system so that any instance that can be recovered
        // when the Activity is recreated.
        savedInstanceState.putInt(SOME_VALUE, someIntValue);
        super.onSaveInstanceState(savedInstanceState);
    }

    public void onRestoreInstanceState(Bundle savedInstanceState) {
        // This is called by the system when the Activity is reconstructed.
        super.onRestoreInstanceState(savedInstanceState);
        someIntValue = savedInstanceState.getInt(SOME_VALUE);
    }
}
```

There are other lifecycle events where state can be restored such as in `Activity#onCreate()`. If you want to learn more about how this works, check out [CodePath’s wiki](https://guides.codepath.com/android/Handling-Configuration-Changes).

You might be saying, why not declare in the AndroidManifest that the Activity handles orientation configuration changes so that state will be retained automatically?

i.e. 

```
<activity android:name=”com.app.MyActivity”
          android:configChanges=”orientation|screenSize” />

```

Although there are some valid use cases where you want to do this, this should be done as a last resort. This is a very common beginner mistake as it appears to get around the issue of saving state; however, doing so might have some unintended consequences. For example, say you want to declare a landscape/portrait-specific resource, that resource will not be loaded automatically on a configuration change and you need to instead explicitly load the resource in `Activity#onConfigurationChanged()`. Not knowing this consequence can be a pain to debug.

A quote from [Romain Guy](http://www.curious-creature.com/):

> ...it is sometimes confusing for new Android developers who wonder why their activity is destroyed and recreated. Facing this “issue,” some developers choose to handle configuration changes themselves which is, in my opinion, a short-term solution that will complicate their life when other devices come out or when the application becomes more complex. The automatic resource handling is a very efficient and easy way to adapt your application’s user interface to various devices and devices configurations.

Indeed, it is good practice to allow the system to do what it was designed to do as it allows your application to behave correctly on varying devices especially as your application gets more complex.

In conclusion, locking the device on a particular orientation—or letting the Activity handle orientation configuration changes—is a stopgap solution without taking into consideration how configuration changes work. Develop apps as if the user may drop off from the screen that they are currently on and ensure that state is appropriately saved/restored.

#### TL;DR
 * if it’s purely a UX reason for locking orientation, do so but be cautious that the app still handles configuration changes properly.
 
 * if `configChanges=”orientation”` is added, make sure it’s actually needed (e.g. for performance reasons maybe because it’s really expensive to reconstruct the Activity, etc.).
 
 * if the above solutions are used merely to “solve” saving state on an orientation change, don’t do it.
