---
layout: post
title: "Reactive Modelling on Android"
date: 2017-03-06 13:14:25 +0100
comments: true
categories: Android, RxJava
---

_This post was originally published on [Toptal](https://www.toptal.com/android/simplify-concurrency-reactive-modelling-android?utm_source=Android+Weekly&utm_campaign=258130a563-android-weekly-247&utm_medium=email&utm_term=0_4eb677ad19-258130a563-337833853)._

Concurrency and asynchronicity are inherent to mobile programming.

Dealing with concurrency through imperative-style programming, which is what programming on Android generally involves, can be the cause of many problems. Using Reactive Programming with [RxJava](https://github.com/ReactiveX/RxJava), you can avoid potential concurrency problems by providing a cleaner and less error-prone solution.

Aside from simplifying concurrent, asynchronous tasks, RxJava also provides the ability to perform functional style operations that transform, combine, and aggregate emissions from an Observable until we achieve our desired result.

{% img center http://chrisarriola.me/images/rxjava_3os.jpg %}

By combining RxJava’s reactive paradigm and functional style operations, we can model a wide range of concurrency constructs in a reactive way, even in Android’s non-reactive world. In this article, you will learn how you can do exactly that. You will also learn how to adopt RxJava into an existing project incrementally.

If you are new to RxJava, I recommend reading the post [here](https://www.toptal.com/android/functional-reactive-android-rxjava) which talks about some of the fundamentals of RxJava.

## Bridging Non-Reactive into the Reactive World

One of the challenges of adding RxJava as one of the libraries to your project is that it fundamentally changes the way that you reason about your code.

RxJava requires you to think about data as being pushed rather than being pulled. While the concept itself is simple, changing a full codebase that is based on a pull paradigm can be a bit daunting. Although consistency is always ideal, you might not always have the privilege to make this transition throughout your entire code base all at once, so more of an incremental approach may be needed.

Consider the following code:
```
/**
 * @return a list of users with blogs
 */
public List<User> getUsersWithBlogs() {
   final List<User> allUsers = UserCache.getAllUsers();
   final List<User> usersWithBlogs = new ArrayList<>();
   for (User user : allUsers) {
       if (user.blog != null && !user.blog.isEmpty()) {
           usersWithBlogs.add(user);
       }
   }
   Collections.sort(usersWithBlogs, (user1, user2) -> user1.name.compareTo(user2.name));
   return usersWithBlogs;
}
```

This function gets a list of `User` objects from the cache, filters each one based on whether or not the user has a blog, sorts them by the user’s name, and finally returns them to the caller. Looking at this snippet, we notice that many of these operations can take advantage of RxJava operators; e.g., `filter()` and `sorted()`.

Rewriting this snippet then gives us:

```
/**
 * @return a list of users with blogs
 */
public Observable<User> getUsersWithBlogs() {
   return Observable.fromIterable(UserCache.getAllUsers())
                    .filter(user -> user.blog != null && !user.blog.isEmpty())
                    .sorted((user1, user2) -> user1.name.compareTo(user2.name));
}
```

The first line of the function converts the `List<User>` returned by `UserCache.getAllUsers()` to an `Observable<User>` via `fromIterable()`. This is the first step into making our code reactive. Now that we are operating on an `Observable`, this enables us to perform any `Observable` operator in the RxJava toolkit – `filter()` and `sorted()` in this case.

There are a few other points to note about this change.

First, the method signature is no longer the same. This may not be a huge deal if this method call is only used in a few places and it’s easy to propagate the changes up to other areas of the stack; however, if it breaks clients relying on this method, that is problematic and the method signature should be reverted.

Second, RxJava is designed with laziness in mind. That is, no long operations should be performed when there are no subscribers to the `Observable`. With this modification, that assumption is no longer true since `UserCache.getAllUsers()` is invoked even before there are any subscribers.

## Leaving the Reactive World

To address the first issue from our change, we can make use of any of the blocking operators available to an `Observable` such as `blockingFirst()` and `blockingNext()`. Essentially, both of these operators will block until an item is emitted downstream: `blockingFirst()` will return the first element emitted and finish, whereas `blockingNext()` will return an `Iterable` which allows you to perform a for-each loop on the underlying data (each iteration through the loop will block).

A side-effect of using a blocking operation that is important to be aware of, though, is that exceptions are thrown on the calling thread rather than being passed to an observer’s `onError()` method.

Using a blocking operator to change the method signature back to a `List<User>`, our snippet would now look like this:

```
/**
 * @return a list of users with blogs
 */
public List<User> getUsersWithBlogs() {
   return Observable.fromIterable(UserCache.getAllUsers())
           .filter(user -> user.blog != null && !user.blog.isEmpty())
           .sorted((user1, user2) -> user1.name.compareTo(user2.name))
           .toList()
           .blockingGet();
}
```

Before calling a blocking operator (i.e. `blockingGet()`) we first need to chain the aggregate operator `toList()` so that the stream is modified from an `Observable<User>` to a `Single<List<User>>` (a Single is a special type of Observable that only emits a single value in `onSuccess()`, or an error via `onError()`).

Afterwards, we can call the blocking operator `blockingGet()` which unwraps the `Single` and returns a `List<User>`.

Although RxJava supports this, as much as possible this should be avoided as this is not idiomatic reactive programming. When absolutely necessary though, blocking operators are a nice initial way of stepping out of the reactive world.

## The Lazy Approach

As mentioned earlier, RxJava was designed with laziness in mind. That is, long-running operations should be delayed as long as possible (i.e., until a subscribe is invoked on an `Observable`). To make our solution lazy, we make use of the `defer()` operator.

{% img center http://chrisarriola.me/images/rxjava_defer.jpg %}

`defer()` takes in an `ObservableSource` factory which creates an `Observable` for each new observer that subscribes. In our case, we want to return `Observable.fromIterable(UserCache.getAllUser())` whenever an observer subscribes.

```
/**
 * @return a list of users with blogs
 */
public Observable<User> getUsersWithBlogs() {
   return Observable.defer(() -> Observable.fromIterable(UserCache.getAllUsers()))
                    .filter(user -> user.blog != null && !user.blog.isEmpty())
                    .sorted((user1, user2) -> user1.name.compareTo(user2.name));
}
```

Now that the long running operation is wrapped in a `defer()`, we have full control as to what thread this should run in simply by specifying the appropriate `Scheduler` in `subscribeOn()`. With this change, our code is fully reactive and subscription should only occur at the moment the data is needed.

```
/**
 * @return a list of users with blogs
 */
public Observable<User> getUsersWithBlogs() {
   return Observable.defer(() -> Observable.fromIterable(UserCache.getAllUsers()))
                    .filter(user -> user.blog != null && !user.blog.isEmpty())
                    .sorted((user1, user2) -> user1.name.compareTo(user2.name))
                    .subscribeOn(Schedulers.io());
}
```

Another quite useful operator for deferring computation is the `fromCallable()` method. Unlike `defer()`, which expects an Observable to be returned in the lambda function and in turn “flattens” the returned `Observable`, `fromCallable()` will invoke the lambda and return the value downstream.

```
/**
 * @return a list of users with blogs
 */
public Observable<User> getUsersWithBlogs() {
   final Observable<List<User>> usersObservable = Observable.fromCallable(() -> UserCache.getAllUsers());
   final Observable<User> userObservable = usersObservable.flatMap(users -> Observable.fromIterable(users));
   return userObservable.filter(user -> user.blog != null && !user.blog.isEmpty())
                        .sorted((user1, user2) -> user1.name.compareTo(user2.name));
}
```

Single using `fromCallable()` on a list would now return an `Observable<List<User>>`, we need to flatten this list using `flatMap()`.

## Reactive-everything

From the previous examples, we have seen that we can wrap any object in an `Observable` and jump between non-reactive and reactive states using blocking operations and `defer()`/`fromCallable()`. Using these constructs, we can start converting areas of an Android app to be reactive.


### Long Operations

A good place to initially think of using RxJava is whenever you have a process that takes a while to perform, such as network calls (check out [previous post](https://www.toptal.com/android/functional-reactive-android-rxjava) for examples), disk reads and writes, etc. The following example illustrates a simple function that will write text to the file system:

```
/**
 * Writes {@code text} to the file system.
 *
 * @param context a Context
 * @param filename the name of the file
 * @param text the text to write
 * @return true if the text was successfully written, otherwise, false
 */
public boolean writeTextToFile(Context context, String filename, String text) {
   FileOutputStream outputStream;
   try {
       outputStream = context.openFileOutput(filename, Context.MODE_PRIVATE);
       outputStream.write(text.getBytes());
       outputStream.close();
       return true;
   } catch (Exception e) {
       e.printStackTrace();
       return false;
   }
}
```

When calling this function, we need to make sure that it is done on a separate thread since this operation is blocking. Imposing such a restriction on the caller complicates things for the developer which increases the likelihood of bugs and can potentially slow down development.

Adding a comment to the function will of course help avoid errors by the caller, but that is still far from bulletproof.

Using RxJava, however, we can easily wrap this into an `Observable` and specify the `Scheduler` that it should run on. This way, the caller doesn’t need to be concerned at all with invoking the function in a separate thread; the function will take care of this itself.

```
/**
 * Writes {@code text} to the filesystem.
 *
 * @param context a Context
 * @param filename the name of the file
 * @param text the text to write
 * @return An Observable emitting a boolean indicating whether or not the text was successfully written.
 */
public Observable<Boolean> writeTextToFile(Context context, String filename, String text) {
   return Observable.fromCallable(() -> {
       FileOutputStream outputStream;
       outputStream = context.openFileOutput(filename, Context.MODE_PRIVATE);
       outputStream.write(text.getBytes());
       outputStream.close();
       return true;
   }).subscribeOn(Schedulers.io());
}
```

Using `fromCallable()`, writing the text to file is deferred up until subscription time.

Since exceptions are first-class objects in RxJava, one other benefit of our change is that the we no longer need to wrap the operation in a try/catch block. The exception will simply be propagated downstream rather than being swallowed. This allows the caller to handle the exception a he/she sees fit (e.g. show an error to the user depending on what exception was thrown, etc.).

One other optimization we can perform is to return a `Completable` rather than an `Observable`. A `Completable` is essentially a special type of `Observable` — similar to a `Single` — that simply indicates if a computation succeeded, via `onComplete()`, or failed, via `onError()`. Returning a `Completable` seems to make more sense in this case since it seems silly to return a single true in an `Observable` stream.

```
/**
 * Writes {@code text} to the filesystem.
 *
 * @param context a context
 * @param filename the name of the file
 * @param text the text to write
 * @return A Completable
 */
public Completable writeTextToFile(Context context, String filename, String text) {
   return Completable.fromAction(() -> {
       FileOutputStream outputStream;
       outputStream = context.openFileOutput(filename, Context.MODE_PRIVATE);
       outputStream.write(text.getBytes());
       outputStream.close();
   }).subscribeOn(Schedulers.io());
}
```

To complete the operation, we use the `fromAction()` operation of a `Completable` since the return value is no longer of interest to us. If needed, like an `Observable`, a `Completable` also supports the `fromCallable()` and `defer()` functions.

### Replacing Callbacks

So far, all the examples that we’ve looked at emit either one value (i.e., can be modelled as a `Single`), or tell us if an operation succeeded or failed (i.e., can be modelled as a `Completable`).

How might we convert areas in our app, though, that receive continuous updates or events (such as location updates, view click events, sensor events, and so on)?

We will look at two ways to do this, using `create()` and using `Subjects`.

`create()` allows us to explicitly invoke an observer’s `onNext()`|`onComplete()`|`onError()` method as we receive updates from our data source. To use `create()`, we pass in an `ObservableOnSubscribe` which receives an `ObservableEmitter` whenever an observer subscribes. Using the received emitter, we can then perform all the necessary set-up calls to start receiving updates and then invoke the appropriate `Emitter` event.

In the case of location updates, we can register to receive updates in this place and emit location updates as received.

```
public class LocationManager {

   /**
    * Call to receive device location updates.
    * @return An Observable emitting location updates
    */
   public Observable<Location> observeLocation() {
       return Observable.create(emitter -> {
           // Make sure that the following conditions apply and if not, call the emitter's onError() method
           // (1) googleApiClient is connected
           // (2) location permission is granted
           final LocationRequest locationRequest = new LocationRequest();
           locationRequest.setInterval(1000);
           locationRequest.setPriority(LocationRequest.PRIORITY_HIGH_ACCURACY);

           LocationServices.FusedLocationApi.requestLocationUpdates(googleApiClient, locationRequest, new LocationListener() {
               @Override public void onLocationChanged(Location location) {
                   if (!emitter.isDisposed()) {
                       emitter.onNext(location);
                   }
               }
           });
       });
   }
}
```

The function inside the `create()` call requests location updates and passes in a callback that gets invoked when the device’s location changes. As we can see here, we essentially replace the callback-style interface and instead emit the received location in the created Observable stream (for the sake of educational purposes, I skipped some of the details with constructing a location request, if you want to delve deeper into the details you can read it [here](https://developer.android.com/training/location/receive-location-updates.html)).

One other thing to note about `create()` is that, whenever `subscribe()` is called, a new emitter is provided. In other words, `create()` returns a [cold Observable](https://medium.com/@benlesh/hot-vs-cold-observables-f8094ed53339). This means that, in the function above, we would potentially be requesting location updates multiple times, which is not what we want.

To work around this, we want to change the the function to return a hot `Observable` with the help of `Subjects`.

### Enter Subjects

A `Subject` extends an `Observable` and implements `Observer` at the same time. This is particularly useful whenever we want to emit or cast the same event to multiple subscribers at the same time. Implementation-wise, we would want to expose the `Subject` as an `Observable` to clients, while keeping it as a `Subject` for the provider.

```
public class LocationManager {

   private Subject<Location> locationSubject = PublishSubject.create();
   
   /**
    * Invoke this method when this LocationManager should start listening to location updates.
    */
   public void connect() {
       final LocationRequest locationRequest = new LocationRequest();
       locationRequest.setInterval(1000);
       locationRequest.setPriority(LocationRequest.PRIORITY_HIGH_ACCURACY);

       LocationServices.FusedLocationApi.requestLocationUpdates(googleApiClient, locationRequest, new LocationListener() {
           @Override public void onLocationChanged(Location location) {
               locationSubject.onNext(location);
           }
       });
   }
   
   /**
    * Call to receive device location updates.
    * @return An Observable emitting location updates
    */
   public Observable<Location> observeLocation() {
       return locationSubject;
   }
}
```

In this new implementation, the subtype `PublishSubject` is used which emits events as they arrive starting from the time of subscription. Accordingly, if a subscription is performed at a point when location updates have already been emitted, past emissions will not be received by the observer, only subsequent ones. If this behavior is not desired, there are a couple of other `Subject` subtypes in the RxJava toolkit that can be [used](http://reactivex.io/documentation/subject.html).

{% img center http://chrisarriola.me/images/rxjava_publish_subject.jpg %}

In addition, we also created a separate `connect()` function which starts the request to receive location updates. The `observeLocation()` can still do the `connect()` call, but we refactored it out of the function for clarity/simplicity.

## Summary

We’ve looked at a number of mechanisms and techniques:

* `defer()` and its variants to delay execution of a computation until subscription
* cold `Observables` generated through `create()`
* hot `Observables` using `Subjects`
* `blockingX()` operations when we want to leave the reactive world

Hopefully, the examples provided in this article inspired some ideas regarding different areas in your app that can be converted to be reactive. We’ve covered a lot and if you have any questions, suggestions, or if anything is not clear, feel free to leave a comment below!

If you are interested in learning more about RxJava, I am working on an in-depth book that explains how to view problems the reactive way using Android examples. If you’d like to receive updates on it, please subscribe [here](https://leanpub.com/reactiveandroid).

{% img center http://chrisarriola.me/images/rxjava_my_book.jpg %}