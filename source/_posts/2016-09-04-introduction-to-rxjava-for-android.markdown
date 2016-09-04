---
layout: post
title: "Introduction to RxJava for Android"
date: 2016-09-04 15:52:08 +0800
comments: true
categories: Android, RxJava
---

_This post was originally published on [Toptal](https://www.toptal.com/android/functional-reactive-android-rxjava) under my Toptal [account](https://www.toptal.com/resume/christopher-arriola)._

If you’re an Android developer, chances are you’ve heard of [RxJava](https://github.com/ReactiveX/RxJava). It’s one of the most discussed libraries for enabling Functional Reactive Programming (FRP) in Android development. It’s touted as the go-to framework for simplifying concurrency/asynchronous tasks inherent in mobile programming.

But… what is RxJava and how does it “simplify” things?

{% img center http://chrisarriola.me/images/rxjava_ooo.jpg %}

While there are lots of resources already available online explaining what RxJava is, in this article my goal is to give you a basic introduction to RxJava and specifically how it fits into Android development. I’ll also give some concrete examples and suggestions on how you can integrate it in a new or existing project.

### Why Consider RxJava

At its core, RxJava simplifies development because it [raises the level of abstraction](http://reactivex.io/intro.html) around threading. That is, as a developer you don’t have to worry too much about the details of how to perform operations that should occur on different threads. This is particularly attractive since threading is challenging to get right and, if not correctly implemented, can cause some of the most difficult bugs to debug and fix.

Granted, this doesn’t mean RxJava is bulletproof when it comes to threading and it is still important to understand what’s happening behind the scenes; however, RxJava can definitely make your life easier.

Let’s look at an example.

#### Network Call - RxJava vs AsyncTask

Say we want to obtain data over the network and update the UI as a result. One way to do this is to (1) create an inner `AsyncTask` subclass in our `Activity`/`Fragment`, (2) perform the network operation in the background, and (3) take the result of that operation and update the UI in the main thread.

```
public class NetworkRequestTask extends AsyncTask<Void, Void, User> {

    private final int userId;

    public NetworkRequestTask(int userId) {
        this.userId = userId;
    }

    @Override protected User doInBackground(Void... params) {
        return networkService.getUser(userId);
    }

    @Override protected void onPostExecute(User user) {
        nameTextView.setText(user.getName());
        // ...set other views
    }
}
   
private void onButtonClicked(Button button) {
   new NetworkRequestTask(123).execute()
}
```

Harmless as this may seem, this approach has some issues and limitations. Namely, memory/context leaks are easily created since `NetworkRequestTask` is an inner class and thus holds an implicit reference to the outer class. Also, what if we want to chain another long operation after the network call? We’d have to nest two `AsyncTask`s which can significantly reduce readability.

In contrast, an RxJava approach to performing a network call might look something like this:

```
private Subscription subscription;

private void onButtonClicked(Button button) {
   subscription = networkService.getObservableUser(123)
                 .subscribeOn(Schedulers.io())
                 .observeOn(AndroidSchedulers.mainThread())
                 .subscribe(new Action1<User>() {
                     @Override public void call(User user) {
                         nameTextView.setText(user.getName());
                         // ... set other views
                     }
                 });
}

@Override protected void onDestroy() {
   if (subscription != null && !subscription.isUnsubscribed()) {
       subscription.unsubscribe();
   }
   super.onDestroy();
}
```

Using this approach, we solve the problem (of potential memory leaks caused by a running thread holding a reference to the outer context) by keeping a reference to the returned `Subscription` object. This `Subscription` object is then tied to the `Activity`/`Fragment` object’s `#onDestroy()` method to guarantee that the `Action1#call` operation does not execute when the `Activity`/`Fragment` needs to be destroyed.

Also, notice that that the return type of `#getObservableUser(...)` (i.e. an `Observable<User>`) is chained with further calls to it. Through this fluid API, we’re able to solve the second issue of using an `AsyncTask` which is that it allows further network call/long operation chaining. Pretty neat, huh?

Let’s dive deeper into some RxJava concepts.

### Observable, Observer, and Operator - The 3 O’s of RxJava Core

In the RxJava world, everything can be modeled as streams. A stream emits item(s) over time, and each emission can be consumed/observed.

If you think about it, a stream is not a new concept: click events can be a stream, location updates can be a stream, push notifications can be a stream, and so on.

{% img http://chrisarriola.me/images/rxjava_animation.gif %}

The stream abstraction is implemented through 3 core constructs which I like to call “the 3 O’s”; namely: the **O**bservable, **O**bserver, and the **O**perator. The **Observable** emits items (the stream); and the **Observer** consumes those items. Emissions from Observable objects can further be modified, transformed, and manipulated by chaining **Operator** calls.

#### Observable

An Observable is the stream abstraction in RxJava. It is similar to an **Iterator** in that, given a sequence, it iterates through and produces those items in an orderly fashion. A consumer can then consume those items through the same interface, regardless of the underlying sequence.

Say we wanted to emit the numbers 1, 2, 3, in that order. To do so, we can use the `Observable<T>#create(OnSubscribe<T>)` method.

```
Observable<Integer> observable = Observable.create(new Observable.OnSubscribe<Integer>() {
   @Override public void call(Subscriber<? super Integer> subscriber) {
       subscriber.onNext(1);
       subscriber.onNext(2);
       subscriber.onNext(3);
       subscriber.onCompleted();
   }
});
```

Invoking `subscriber.onNext(Integer)` emits an item in the stream and, when the stream is finished emitting, `subscriber.onCompleted()` is then invoked.

This approach to creating an Observable is fairly verbose. For this reason, there are convenience methods for creating Observable instances which should be preferred in almost all cases.

The simplest way to create an Observable is using `Observable#just(...)`. As the method name suggests, it just emits the item(s) that you pass into it as method arguments.

```
Observable.just(1, 2, 3); // 1, 2, 3 will be emitted, respectively
```

#### Observer

The next component to the Observable stream is the Observer (or Observers) subscribed to it. Observers are notified whenever something “interesting” happens in the stream. Observers are notified via the following events:

* `Observer#onNext(T)` - invoked when an item is emitted from the stream
* `Observable#onError(Throwable)` - invoked when an error has occurred within the stream
* `Observable#onCompleted()` - invoked when the stream is finished emitting items.

To subscribe to a stream, simply call Observable<T>#subscribe(...) and pass in an Observer instance.

```
Observable<Integer> observable = Observable.just(1, 2, 3);
observable.subscribe(new Observer<Integer>() {
   @Override public void onCompleted() {
       Log.d("Test", "In onCompleted()");
   }

   @Override public void onError(Throwable e) {
       Log.d("Test", "In onError()");
   }

   @Override public void onNext(Integer integer) {
       Log.d("Test", "In onNext():" + integer);
   }
});
```

The above code will emit the following in Logcat:

```
In onNext(): 1
In onNext(): 2
In onNext(): 3
In onNext(): 4
In onCompleted()
```

There may also be some instances where we are no longer interested in the emissions of an Observable. This is particularly relevant in Android when, for example, an `Activity`/`Fragment` needs to be reclaimed in memory.

To stop observing items, we simply need to call `Subscription#unsubscribe()` on the returned Subscription object.

```
Subscription subscription = someInfiniteObservable.subscribe(new Observer<Integer>() {
   @Override public void onCompleted() {
       // ...
   }

   @Override public void onError(Throwable e) {
       // ...
   }

   @Override public void onNext(Integer integer) {
       // ...
   }
});

// Call unsubscribe when appropriate
subscription.unsubscribe();
```

As seen in the code snippet above, upon subscribing to an Observable, we hold the reference to the returned Subscription object and later invoke `subscription#unsubscribe()` when necessary. In Android, this is best invoked within `Activity#onDestroy()` or `Fragment#onDestroy()`.

#### Operator

Items emitted by an Observable can be transformed, modified, and filtered through Operators before notifying the subscribed Observer object(s). Some of the most common operations found in functional programming (such as map, filter, reduce, etc.) can also be applied to an Observable stream. Let’s look at map as an example:

```
Observable.just(1, 2, 3, 4, 5).map(new Func1<Integer, Integer>() {
   @Override public Integer call(Integer integer) {
       return integer * 2;
   }
}).subscribe(new Observer<Integer>() {
   @Override public void onCompleted() {
       // ...
   }

   @Override public void onError(Throwable e) {
       // ...
   }

   @Override public void onNext(Integer integer) {
       // ...
   }
});
```

The code snippet above would take each emission from the Observable and multiply each by 2, producing the stream 2, 4, 6, 8, 10, respectively. Applying an Operator typically returns another Observable as a result, which is convenient as this allows us to chain multiple operations to obtain a desired result.

Given the stream above, say we wanted to only receive even numbers. This can be achieved by chaining a _filter_ operation.

```
Observable.just(1, 2, 3, 4, 5).map(new Func1<Integer, Integer>() {
   @Override public Integer call(Integer integer) {
       return integer * 2;
   }
}).filter(new Func1<Integer, Boolean>() {
   @Override public Boolean call(Integer integer) {
       return integer % 2 == 0;
   }
}).subscribe(new Observer<Integer>() {
   @Override public void onCompleted() {
       // ...
   }

   @Override public void onError(Throwable e) {
       // ...
   }

   @Override public void onNext(Integer integer) {
       // ...
   }
});
```

There are [many operators built-in the RxJava](http://reactivex.io/documentation/operators.html#alphabetical) toolset that modify the Observable stream; if you can think of a way to modify the stream, chances are, there’s an Operator for it. Unlike most technical documentation, reading the RxJava/ReactiveX docs is fairly simple and to-the-point. Each operator in the documentation comes along with a visualization on how the Operator affects the stream. These visualizations are called “marble diagrams.”

Here’s how a hypothetical Operator called flip might be modeled through a marble diagram:

{% img http://chrisarriola.me/images/rxjava_flip.png %}

### Multithreading with RxJava

Controlling the thread within which operations occur in the Observable chain is done by specifying the [Scheduler](http://reactivex.io/documentation/scheduler.html) within which an operator should occur. Essentially, you can think of a Scheduler as a thread pool that, when specified, an operator will use and run on. By default, if no such Scheduler is provided, the Observable chain will operate on the same thread where `Observable#subscribe(...)` is called. Otherwise, a Scheduler can be specified via `Observable#subscribeOn(Scheduler)` and/or `Observable#observeOn(Scheduler)` wherein the scheduled operation will occur on a thread chosen by the Scheduler.

The key difference between the two methods is that `Observable#subscribeOn(Scheduler)` instructs the source Observable which Scheduler it should run on. The chain will continue to run on the thread from the Scheduler specified in `Observable#subscribeOn(Scheduler)` until a call to `Observable#observeOn(Scheduler)` is made with a different Scheduler. When such a call is made, all observers from there on out (i.e., subsequent operations down the chain) will receive notifications in a thread taken from the observeOn Scheduler.

Here’s a marble diagram that demonstrates how these methods affect where operations are run:

{% img http://chrisarriola.me/images/rxjava_schedulers.png %}

In the context of Android, if a UI operation needs to take place as a result of a long operation, we’d want that operation to take place on the UI thread. For this purpose, we can use `AndroidScheduler#mainThread()`, one of the Schedulers provided in the [RxAndroid](https://github.com/ReactiveX/RxAndroid) library.

### RxJava on Android

Now that we’ve got some of the basics under our belt, you might be wondering — what’s the best way to integrate RxJava in an [Android](https://www.toptal.com/android) application? As you might imagine, there are many use cases for RxJava but, in this example, let’s take a look at one specific case: using Observable objects as part of the network stack.

In this example, we will look at [Retrofit](http://square.github.io/retrofit/), an HTTP client open sourced by Square which has built-in bindings with RxJava to interact with GitHub’s API. Specifically, we’ll create a simple app that presents all the starred repositories for a user given a GitHub username. If you want to jump ahead, the source code is available [here](https://github.com/arriolac/GitHubRxJava).

#### Create a New Android Project

* Start by creating a new Android project and naming it **GitHubRxJava**.

{% img http://chrisarriola.me/images/rxjava_setup1.png %}

* In the **Target Android Devices** screen, keep **Phone and Tablet** selected and set the minimum SDK level of 17. Feel free to set it to a lower/higher API level but, for this example, API level 17 will suffice.

{% img http://chrisarriola.me/images/rxjava_setup2.png %}

* Select **Empty Activity** in the next prompt.

{% img http://chrisarriola.me/images/rxjava_setup3.png %}

* In the last step, keep the Activity Name as **MainActivity** and generate a layout file **activity_main**.

{% img http://chrisarriola.me/images/rxjava_setup4.png %}

#### Project Set-Up

Include [RxJava](https://github.com/ReactiveX/RxJava), [RxAndroid](https://github.com/ReactiveX/RxAndroid), and the [Retrofit](http://square.github.io/retrofit/) library in `app/build.gradle`. Note that including RxAndroid implicitly also includes RxJava. It is best practice, however, to always include those two libraries explicitly since RxAndroid does not always contain the most up-to-date version of RxJava. Explicitly including the latest version of RxJava guarantees use of the most up-to-date version.

```
dependencies {
    compile 'com.squareup.retrofit2:adapter-rxjava:2.1.0'
    compile 'com.squareup.retrofit2:converter-gson:2.1.0'
    compile 'com.squareup.retrofit2:retrofit:2.1.0'
    compile 'io.reactivex:rxandroid:1.2.0'
    compile 'io.reactivex:rxjava:1.1.8'
    // ...other dependencies
}
```

#### Create Data Object

Create the `GitHubRepo` data object class. This class encapsulates a repository in GitHub (the network response contains more data but we’re only interested in a subset of that).

```
public class GitHubRepo {

    public final int id;
    public final String name;
    public final String htmlUrl;
    public final String description;
    public final String language;
    public final int stargazersCount;

    public GitHubRepo(int id, String name, String htmlUrl, String description, String language, int stargazersCount) {
        this.id = id;
        this.name = name;
        this.htmlUrl = htmlUrl;
        this.description = description;
        this.language = language;
        this.stargazersCount = stargazersCount;
    }
}
```

#### Set-Up Retrofit

* Create the `GitHubService` interface. We will pass this interface into Retrofit and Retrofit will create an implementation of `GitHubService`.

```
    public interface GitHubService {
        @GET("users/{user}/starred") Observable<List<GitHubRepo>> getStarredRepositories(@Path("user") String userName);
    }
```

* Create the `GitHubClient` class. This will be the object we will interact with to make network calls from the UI level.
	*  When constructing an implementation of `GitHubService` through Retrofit, we need to pass in an `RxJavaCallAdapterFactory` as the call adapter so that network calls can return Observable objects (passing a call adapter is needed for any network call that returns a result other than a `Call`).
	* We also need to pass in a `GsonConverterFactory` so that we can use [Gson](https://github.com/google/gson) as a way to marshal JSON objects to Java objects.
	
```
    public class GitHubClient {

        private static final String GITHUB_BASE_URL = "https://api.github.com/";

        private static GitHubClient instance;
        private GitHubService gitHubService;

        private GitHubClient() {
            final Gson gson =
                new GsonBuilder().setFieldNamingPolicy(FieldNamingPolicy.LOWER_CASE_WITH_UNDERSCORES).create();
            final Retrofit retrofit = new Retrofit.Builder().baseUrl(GITHUB_BASE_URL)
                                                            .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                                                            .addConverterFactory(GsonConverterFactory.create(gson))
                                                            .build();
            gitHubService = retrofit.create(GitHubService.class);
        }

        public static GitHubClient getInstance() {
            if (instance == null) {
                instance = new GitHubClient();
            }
            return instance;
        }

        public Observable<List<GitHubRepo>> getStarredRepos(@NonNull String userName) {
            return gitHubService.getStarredRepositories(userName);
        }
    }
```

#### Set-Up Layouts

Next, create a simple UI that displays the retrieved repos given an input GitHub username. Create `activity_home.xml` - the layout for our activity - with something like the following:

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <ListView
        android:id="@+id/list_view_repos"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"/>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal">

        <EditText
            android:id="@+id/edit_text_username"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:hint="@string/username"/>

        <Button
            android:id="@+id/button_search"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@string/search"/>

    </LinearLayout>

</LinearLayout>
```

Create `item_github_repo.xml` - the `ListView` item layout for GitHub repository object - with something like the following:

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="6dp">

    <TextView
        android:id="@+id/text_repo_name"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:textSize="24sp"
        android:textStyle="bold"
        tools:text="Cropper"/>

    <TextView
        android:id="@+id/text_repo_description"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:lines="2"
        android:ellipsize="end"
        android:textSize="16sp"
        android:layout_below="@+id/text_repo_name"
        tools:text="Android widget for cropping and rotating an image."/>

    <TextView
        android:id="@+id/text_language"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_below="@+id/text_repo_description"
        android:layout_alignParentLeft="true"
        android:textColor="?attr/colorPrimary"
        android:textSize="14sp"
        android:textStyle="bold"
        tools:text="Language: Java"/>

    <TextView
        android:id="@+id/text_stars"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_below="@+id/text_repo_description"
        android:layout_alignParentRight="true"
        android:textColor="?attr/colorAccent"
        android:textSize="14sp"
        android:textStyle="bold"
        tools:text="Stars: 1953"/>

</RelativeLayout>
```

#### Glue Everything Together

Create a `ListAdapter` that is in charge of binding `GitHubRepo` objects into `ListView` items. The process essentially involves inflating `item_github_repo.xml` into a `View` if no recycled `View` is provided; otherwise, a recycled `View` is reused to prevent overinflating too many `View` objects.

```
public class GitHubRepoAdapter extends BaseAdapter {

    private List<GitHubRepo> gitHubRepos = new ArrayList<>();

    @Override public int getCount() {
        return gitHubRepos.size();
    }

    @Override public GitHubRepo getItem(int position) {
        if (position < 0 || position >= gitHubRepos.size()) {
            return null;
        } else {
            return gitHubRepos.get(position);
        }
    }

    @Override public long getItemId(int position) {
        return position;
    }

    @Override public View getView(int position, View convertView, ViewGroup parent) {
        final View view = (convertView != null ? convertView : createView(parent));
        final GitHubRepoViewHolder viewHolder = (GitHubRepoViewHolder) view.getTag();
        viewHolder.setGitHubRepo(getItem(position));
        return view;
    }

    public void setGitHubRepos(@Nullable List<GitHubRepo> repos) {
        if (repos == null) {
            return;
        }
        gitHubRepos.clear();
        gitHubRepos.addAll(repos);
        notifyDataSetChanged();
    }

    private View createView(ViewGroup parent) {
        final LayoutInflater inflater = LayoutInflater.from(parent.getContext());
        final View view = inflater.inflate(R.layout.item_github_repo, parent, false);
        final GitHubRepoViewHolder viewHolder = new GitHubRepoViewHolder(view);
        view.setTag(viewHolder);
        return view;
    }

    private static class GitHubRepoViewHolder {

        private TextView textRepoName;
        private TextView textRepoDescription;
        private TextView textLanguage;
        private TextView textStars;

        public GitHubRepoViewHolder(View view) {
            textRepoName = (TextView) view.findViewById(R.id.text_repo_name);
            textRepoDescription = (TextView) view.findViewById(R.id.text_repo_description);
            textLanguage = (TextView) view.findViewById(R.id.text_language);
            textStars = (TextView) view.findViewById(R.id.text_stars);
        }

        public void setGitHubRepo(GitHubRepo gitHubRepo) {
            textRepoName.setText(gitHubRepo.name);
            textRepoDescription.setText(gitHubRepo.description);
            textLanguage.setText("Language: " + gitHubRepo.language);
            textStars.setText("Stars: " + gitHubRepo.stargazersCount);
        }
    }
}
```

Glue everything together in `MainActivity`. This is essentially the `Activity` that gets displayed when we first launch the app. In here, we ask the user to enter their GitHub username, and finally, display all the starred repositories by that username.

```
public class MainActivity extends AppCompatActivity {

    private static final String TAG = MainActivity.class.getSimpleName();
    private GitHubRepoAdapter adapter = new GitHubRepoAdapter();
    private Subscription subscription;

    @Override protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        final ListView listView = (ListView) findViewById(R.id.list_view_repos);
        listView.setAdapter(adapter);

        final EditText editTextUsername = (EditText) findViewById(R.id.edit_text_username);
        final Button buttonSearch = (Button) findViewById(R.id.button_search);
        buttonSearch.setOnClickListener(new View.OnClickListener() {
            @Override public void onClick(View v) {
                final String username = editTextUsername.getText().toString();
                if (!TextUtils.isEmpty(username)) {
                    getStarredRepos(username);
                }
            }
        });
    }

    @Override protected void onDestroy() {
        if (subscription != null && !subscription.isUnsubscribed()) {
            subscription.unsubscribe();
        }
        super.onDestroy();
    }

    private void getStarredRepos(String username) {
        subscription = GitHubClient.getInstance()
                                   .getStarredRepos(username)
                                   .subscribeOn(Schedulers.io())
                                   .observeOn(AndroidSchedulers.mainThread())
                                   .subscribe(new Observer<List<GitHubRepo>>() {
                                       @Override public void onCompleted() {
                                           Log.d(TAG, "In onCompleted()");
                                       }

                                       @Override public void onError(Throwable e) {
                                           e.printStackTrace();
                                           Log.d(TAG, "In onError()");
                                       }

                                       @Override public void onNext(List<GitHubRepo> gitHubRepos) {
                                           Log.d(TAG, "In onNext()");
                                           adapter.setGitHubRepos(gitHubRepos);
                                       }
                                   });
    }
}
```

#### Run the App

Running the app should present a screen with an input box to enter a GitHub username. Searching should then present the list of all starred repos.

{% img http://chrisarriola.me/images/rxjava_screenshot.png %}

#### Conclusion

I hope this serves as a useful introduction to RxJava and an overview of its basic capabilities. There are a ton of powerful concepts in RxJava and I urge you to explore them by digging more deeply into the well-documented [RxJava wiki](https://github.com/ReactiveX/rxjava/wiki	).

Feel free to leave any questions or comments in the comment box below. You can also follow me on Twitter at [@arriolachris](http://twitter.com/arriolachris) where I tweet a lot about RxJava and all things Android.