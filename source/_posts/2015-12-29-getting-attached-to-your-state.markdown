---
layout: post
title: "Getting Attached To Your State"
date: 2015-12-29 15:38:18 -0800
comments: true
categories: Android
---

[Configuration changes](http://developer.android.com/guide/topics/resources/runtime-changes.html#HandlingTheChange) occur in runtime and are caused by various events such as: when keyboard visibility changes, when language changes, or when orientation changes. This in turn causes any *visible Activity* to be reconstructed once the change finishes. For state to be recovered, it needs to be explicitly tied to Android’s parcelling mechanism via Bundle.

### Saving State, The Wrong Way

Upon encountering state loss caused by a configuration change, say on an orientation change, 2 poor solutions are:

#### 1. Locking the orientation mode

```
    <activity 
        android:name="com.app.app.MainActivity"
        android:screenOrientation="portrait" />
```

This is an anti-solution and simply prevents the configuration change from occurring. This will not guard against [the other 13 configuration changes](http://developer.android.com/guide/topics/manifest/activity-element.html#config) as of API level 23.

> Orientation should only be locked as a result of a UI/UX decision, and not for retaining state.

#### 2. Handling configuration changes manually

```
    <activity 
        android:name="com.app.app.MainActivity"
        android:configChanges="orientation" />
```

Again, this is a band-aid and not a proper solution. What if new configuration changes are introduced? Also this might cause some unintended consequences. Say for example you want to declare a landscape/portrait-specific resource, that resource will not be loaded automatically anymore and you need to explicitly load and apply it in **Activity#onConfigurationChanged()** instead. Not knowing this consequence may be a pain to debug.

Quoting [Roman Guy](http://www.curious-creature.com/):

> “...it is sometimes confusing for new Android developers who wonder why their activity is destroyed and recreated. Facing this “issue,” some developers choose to handle configuration changes themselves which is, in my opinion, a short-term solution that will complicate their life when other devices come out or when the application becomes more complex. The automatic resource handling is a very efficient and easy way to adapt your application’s user interface to various devices and devices configurations.”

Indeed, it is good practice to allow the system to do what it was designed to do as it allows your application to behave correctly on varying devices especially as your application gets more complex.

> Manually handling configuration changes should only be done sparingly and as a result of some constraint (e.g. performance reasons).

### Saving State, The Right Way

Luckily, framework Views will automatically be saved and recovered for you—*assuming all your Views have unique IDs*. In other instances you need to explicitly retain state.

#### Retaining State In An Activity

```
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
```
    
There are other lifecycle events where state can be restored such as in **Activity#onCreate()**. Retaining state in a Fragment can be done in a similar way via **Fragment#onSaveInstanceState()**. Here’s [a good resource by CodePath](https://guides.codepath.com/android/Handling-Configuration-Changes) for learning more about how this works.

#### Retaining State in A Custom View

Retaining state in a custom View is a bit more involved but nonetheless also fairly straightforward.

```
package com.operator.android;

import android.content.Context;
import android.os.Parcel;
import android.os.Parcelable;
import android.view.View;

/**
 * A custom {@link View} that demonstrates how to save/restore instance state.
 */
public class CustomView extends View {

    private boolean someState;

    public CustomView(Context context) {
        super(context);
    }

    public boolean isSomeState() {
        return someState;
    }

    public void setSomeState(boolean someState) {
        this.someState = someState;
    }

    @Override protected Parcelable onSaveInstanceState() {
        final Parcelable superState = super.onSaveInstanceState();
        final CustomViewSavedState customViewSavedState = new CustomViewSavedState(superState);
        customViewSavedState.someState = this.someState;
        return customViewSavedState;
    }

    @Override protected void onRestoreInstanceState(Parcelable state) {
        final CustomViewSavedState customViewSavedState = (CustomViewSavedState) state;
        setSomeState(customViewSavedState.someState);
        super.onRestoreInstanceState(customViewSavedState.getSuperState());
    }

    private static class CustomViewSavedState extends BaseSavedState {

        boolean someState;

        public static final Parcelable.Creator<CustomViewSavedState> CREATOR = new Creator<CustomViewSavedState>() {
            @Override public CustomViewSavedState createFromParcel(Parcel source) {
                return new CustomViewSavedState(source);
            }

            @Override public CustomViewSavedState[] newArray(int size) {
                return new CustomViewSavedState[size];
            }
        };

        public CustomViewSavedState(Parcelable superState) {
            super(superState);
        }

        private CustomViewSavedState(Parcel source) {
            super(source);
            someState = source.readInt() == 1;
        }

        @Override public void writeToParcel(Parcel out, int flags) {
            super.writeToParcel(out, flags);
            out.writeInt(someState ? 1 : 0);
        }
    }

}
```

### TL;DR

* Don’t save your state the wrong way 😅 Save your state the right way 😁