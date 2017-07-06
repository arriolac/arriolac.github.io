---
layout: post
title: "Using Observables to Render Responsive Lists: An RxJava Case Study - Part 1 of 3"
date: 2017-07-06 9:40:26 -0500
comments: true
categories: Android, RxJava, Freelancing, Pre
---

_This post was originally posted on [Medium](https://medium.com/joinpre/using-observables-to-render-responsive-lists-an-rxjava-case-study-part-1-of-3-53dd83af2c08)._

This is the 1st part of a 3 part series about how RxJava is used in [Pre](https://pre.co/), a location-based app for checking in and chatting with your best friends. In this first post, I will go over how we used Observables to compose a complex view that displays a list of items, specifically, the _dashboard_ view.

If you are new to RxJava, I recommend [starting here](http://chrisarriola.me/blog/2016/09/04/introduction-to-rxjava-for-android/). You can also check out a [book](https://leanpub.com/reactiveandroid) Angus Huang and I wrote if you would like a more comprehensive learning resource on RxJava.

Note that all of the code samples in this series are written in Kotlin. In addition to RxJava, we use RxKotlin which is a lightweight library that provides convenient extension functions to RxJava.

## Dashboard View

{% img center http://chrisarriola.me/images/pre_dashboard_view.png %}

The dashboard view is the first view presented to the user upon logging in. It contains 2 sections: (1) a section displaying a list of recent check-ins from your friends, and (2) a section displaying a list of all your friends. The latter is displayed by querying the data layer for all of your friends, followed by querying the most recent message in the conversation thread for each of your friends. The state of the message will then determine if an unread indicator, or if a relative timestamp of when that message was sent, should be displayed.

Friend with unread message:
{% img center http://chrisarriola.me/images/pre_dashboard_unread.png %}

Friend with no unread message:
{% img center http://chrisarriola.me/images/pre_dashboard_read.png %}

Both the _Friend_ and _Message_ models are persisted in a local SQLite database. To prevent jankiness while scrolling, retrieving these models should be done before rendering the list. More specifically, we want to make sure that we have all of the Friend objects and the corresponding latest Message in memory before setting the list of friends to the Adapter of the RecyclerView. 
 
Before we dive into Observables and how RxJava fits into the picture, let’s look at a few classes that compose the dashboard view. Note that even though some of the classes here have been shortened for brevity, the underlying concepts should be the same.

## Non-Reactive Parts
### DashboardFriendViewModel
 
On the dashboard, the _DashboardFriendViewModel_ class is in charge of binding the _Friend_ and _Message_ models to a single friend view. True to the _ViewModel_ design pattern, this class provides us with the benefit of abstracting away model details from the view layer. So if our model changes, we would only need to update the _ViewModel_’s code (i.e. _DashboardFriendViewModel_), and not the view’s code.

```
data class DashboardFriendViewModel(
   val emoji: String,
   val title: String,
   val subtitle: String,
   val hasUnread: Boolean
) {
   companion object Factory {
       @JvmStatic fun create(
           friend: Friend,
           message: Message
       ) : DashboardFriendViewModel {
           // Factory logic goes here...
       }
   }
}
```

### DashboardFriendViewHolder
Using a _DashboardFriendViewModel_, a _DashboardFriendViewHolder_ (a subclass of _RecyclerView.ViewHolder_), can simply update the views it holds by getting the fields on the _DashboardFriendViewModel_.
 
```
class DashboardFriendViewHolder(
   parent: ViewGroup
) : RecyclerView.ViewHolder(
    LayoutInflater.from(parent.context).inflate(
        R.layout.item_dashboard_friend, parent, false
    )
) {
 
   var viewModel: DashboardFriendViewModel? = null
       set(value) {
           field = value
           value?.let {
               textViewEmoji.text = it.emoji
               textViewTitle.text = it.title
               textViewSubtitle.text = it.subtitle
               imageViewUnread.visibility = if (hasUnread) View.VISIBLE else View.GONE
           }
       }
}
```

### DashboardAdapter
The backing adapter for the dashboard RecyclerView, _DashboardAdapter_, holds a list of _DashboardFriendViewModel_ objects and binds each object to a _DashboardFriendViewHolder_ (note that check-ins are excluded in the code sample below).

```
class DashboardAdapter : 
    RecyclerView.Adapter<DashboardFriendViewHolder>() {
    var friendViewModels: List<DashboardFriendViewModel> = listOf()
        set(value) {
            field = value
            notifyDataSetChanged()
        }
    override fun onBindViewHolder(
        holder: DashboardFriendViewHolder, 
        position: Int
    ) {
        holder.viewModel = friendViewModels[position]
    }
    override fun getItemCount(): Int = friendViewModels.size
    override fun onCreateViewHolder(
        parent: ViewGroup, 
        viewType: Int
    ): DashboardFriendViewHolder {
        return DashboardFriendViewHolder(parent)
    }
}
```

With these classes in mind, it should be clear that the role of the Activity is to provide the _DashboardAdapter_ a list of _DashboardFriendViewModel_ objects to construct the list of friends in the dashboard view. In the next section, we will look at the reactive elements that come into play to accomplish this.

## Reactive Parts

Each model that is backed by a SQLite table has a corresponding data access object, or DAO for short, that is in charge of performing CRUD (create, read, update, and delete) operations. The DAO provides the application layer with a higher level abstraction when it needs to access models so that it doesn’t need to know how to perform complex SQLite queries.

### DAO

DAOs in Pre are also designed to be reactive; that is, DAOs can provide data packaged as an Observable so that the requestor can continue to receive updates as the underlying model changes. The details of how these Observables are constructed will be covered in the 2nd part of this series. For now, assume that we have the following methods provided by _FriendDao_ and _MessageDao_.

```
class FriendDao {
    fun getAllFriends(): Observable<List<Friend>> {
        // ...
    }
    fun getFriend(userId: String): Observable<Friend> {
        // ...
    }
    
    // more methods here...
}

class MessageDao {
 
    fun getAllMessagesFrom(userId: String): Observable<List<Message>> {
        // ...
    }
    fun getRecentMessageWith(userId: String): Observable<Message> {   
        // ...
    }
    // more methods here...
}
```

It is important to keep in mind that these methods return Observables that report changes to subscribers as the underlying data also changes. For example, if we subscribe to _FriendDao#getAllFriends()_ and, at a later point, a new _Friend_ is added, the observer would receive an updated list of friends via _.onNext()_ once the new friend is persisted in the database. This is really powerful as the mechanism for requesting initial data and for receiving updates will all be in the same place — the observer.

In addition, all methods that return Observables in Pre’s DAOs run off the main thread. It is up to the subscriber to ultimately hop back to the main thread, via _.observeOn()_, when necessary (e.g. when interacting with the view layer).

## Putting It All Together 🏗

Using _FriendDao_ and _MessageDao_ that supply us with Observables of _Friend_ and _Message_ objects, respectively, we can now construct the list of DashboardFriendViewModel objects and display our list of friends on the dashboard.

The lines of code that accomplish this are:

```
friendDao.getAllFriends()
    .switchMap { friends ->
        val observables = friends.map { friend ->
            messageDao.getRecentMessageWith(friend.userId)
                .map { DashboardFriendViewModel.create(friend, it) }
        }
        Observable.combineLatest(observables) {
            Arrays.copyOf(
                it, 
                it.size,
                Array<DashboardFriendViewModel>::class.java
            ).toList()
        }
    }.observeOn(AndroidSchedulers.mainThread())
     .subscribe { adapter.friendViewModels = it }
```

There’s quite a lot going on here so let’s look at it step-by-step:

* First, we retrieve all friends by calling the .getAllFriends() method on a FriendDao object. As mentioned earlier, this Observable will first emit the list of friends as well as any changes to the user’s friends (i.e. a list of friends will be emitted as new friends are added, or as existing friends are removed/updated)

* Next, we apply a .switchMap() operator which maps the stream from a list of Friend objects to a list of DashboardFriendViewModel objects, the type we are ultimately interested in. We are using .switchMap() here, instead of .flatMap() or .concatMap(), so that only the most recent Observable emission from .switchMap() is observed and all other previous emissions will be considered stale. You can read about the difference between the operators on the ReactiveX wiki.

* Within the .switchMap() operator, we then convert friends into a list of Observable<DashboardFriendViewModel> by iterating through each friend and getting the most recent message with them followed by mapping that to a DashboardFriendViewModel.

* We then combine the emissions of the list of Observable<DashboardFriendViewModel> using the .combineLatest() operator, the result of which is then combined into a list of DashboardFriendViewModel objects which is propagated down to the observer. This operator allows downstream operators and observers to receive updates whenever there is a new message from a friend.

* After the .switchMap() operator, we then chain an .observeOn() operator and make sure that the observer receives events on Android’s main thread.

* Finally, we subscribe to the Observable after applying .switchMap() and set the DashboardAdapter’s friendViewModels to the received emissions. Here, we are replacing the backing data set for the DashboardAdapter, which in turn invokes notifyDataSetChanged(). This can also be optimized by finding the items with changes and selectively applying notifiyItemChanged() or notifyItemRangeChanged().

## Summary
The solution above is brief and accomplishes our goal of displaying the dashboard view as well as keeping it up-to-date whenever there is new data. Having a reactive data layer at Pre enables this. By no means is this the only way to compose complex UI. However, I hope this post convinced you that working with Observables makes dealing with data that can come from different sources as well as coordinating concurrency an easier task.

---
Pre is now available in the Google Play Store 🎉 Get it [here](https://play.google.com/store/apps/details?id=io.tsukemen.zarusoba).

Subscribe at the bottom of this site if you want to be notified when the next post in the series goes out.