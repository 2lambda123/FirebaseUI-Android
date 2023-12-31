# FirebaseUI for Realtime Database

FirebaseUI makes it simple to bind data from the Firebase Realtime Database to your app's UI.

Before using this library, you should be familiar with the following topics:

* [Working with lists of data in Firebase Realtime Database][firebase-lists].
* [Displaying lists of data using a RecyclerView][recyclerview].

## Table of contents

1. [Data model](#data-model)
1. [Querying](#querying)
1. [Populating a RecyclerView](#using-firebaseui-to-populate-a-recyclerview)
   1. [Using the FirebaseRecyclerAdapter](#using-the-firebaserecycleradapter)
   1. [Using the FirebaseRecyclerPagingAdapter](#using-the-firebaserecyclerpagingadapter)
1. [Populating a ListView](#using-firebaseui-to-populate-a-listview)
1. [Handling indexed data](#using-firebaseui-with-indexed-data)
   1. [Warnings](#a-note-on-ordering)

## Data model

Imagine you have a chat app where each chat message is an item in the `chats` node
of your database. In your app, you may represent a chat message like this:

```java
public class Chat {
    private String mName;
    private String mMessage;
    private String mUid;

    public Chat() {}  // Needed for Firebase

    public Chat(String name, String message, String uid) {
        mName = name;
        mMessage = message;
        mUid = uid;
    }

    public String getName() { return mName; }

    public void setName(String name) { mName = name; }

    public String getMessage() { return mMessage; }

    public void setMessage(String message) { mMessage = message; }

    public String getUid() { return mUid; }

    public void setUid(String uid) { mUid = uid; }
}
```

A few things to note about this model class:

* The getters and setters follow the JavaBean naming pattern which allows Firebase to map
  the data to field names (ex: `getName()` provides the `name` field).
* The class has an empty constructor, which is required for Firebase's automatic data mapping.

For a properly constructed model class like the `Chat` class above, Firebase can perform automatic
serialization in `DatabaseReference#setValue()` and automatic deserialization in
`DataSnapshot#getValue()`.

### Querying

On the main screen of your app, you may want to show the 50 most recent chat messages. With Firebase
you would use the following query:

```java
Query query = FirebaseDatabase.getInstance()
        .getReference()
        .child("chats")
        .limitToLast(50);
```

To retrieve this data without FirebaseUI, you might use `addChildEventListener` to listen for
live updates:

```java
ChildEventListener childEventListener = new ChildEventListener() {
    @Override
    public void onChildAdded(DataSnapshot dataSnapshot, String previousChildName) {
        // ...
    }

    @Override
    public void onChildChanged(DataSnapshot dataSnapshot, String previousChildName) {
        // ...
    }

    @Override
    public void onChildRemoved(DataSnapshot dataSnapshot) {
        // ...
    }

    @Override
    public void onChildMoved(DataSnapshot dataSnapshot, String previousChildName) {
        // ...
    }

    @Override
    public void onCancelled(DatabaseError databaseError) {
        // ...
    }
};
query.addChildEventListener(childEventListener);
```

## Using FirebaseUI to populate a `RecyclerView`

If you're displaying a list of data, you likely want to bind the `Chat` objects to a `RecyclerView`.
This means implementing a custom `RecyclerView.Adapter` and coordinating updates with the
`ChildEventListener`.

Fear not, FirebaseUI does all of this for you automatically!

### Choosing an adapter

FirebaseUI offers two types of RecyclerView adapters for the Realtime Database:

  * `FirebaseRecyclerAdapter` — binds a `Query` to a `RecyclerView` and responds to all real-time
    events included items being added, removed, moved, or changed. Best used with small result sets
    since all results are loaded at once.
  * `FirebaseRecyclerPagingAdapter` — binds a `Query` to a `RecyclerView` by loading data in pages. Best
    used with large, static data sets. Real-time events are not respected by this adapter, so it
    will not detect new/removed items or changes to items already loaded.

### Using the FirebaseRecyclerAdapter

The `FirebaseRecyclerAdapter` binds a `Query` to a `RecyclerView`. When data is added, removed,
or changed these updates are automatically applied to your UI in real time.

First, configure the adapter by building `FirebaseRecyclerOptions`. In this case we will continue
with our chat example:

```java
 FirebaseRecyclerOptions<Chat> options =
                new FirebaseRecyclerOptions.Builder<Chat>()
                        .setQuery(query, Chat.class)
                        .build();
```

If you need to customize how your model class is parsed, you can use a custom `SnapshotParser`:

```java
...setQuery(..., new SnapshotParser<Chat>() {
    @NonNull
    @Override
    public Chat parseSnapshot(@NonNull DataSnapshot snapshot) {
        return ...;
    }
});
```

Next create the `FirebaseRecyclerAdapter` object. You should already have a `ViewHolder` subclass
for displaying each item. In this case we will use a custom `ChatHolder` class:

```java
FirebaseRecyclerAdapter adapter = new FirebaseRecyclerAdapter<Chat, ChatHolder>(options) {
    @Override
    public ChatHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        // Create a new instance of the ViewHolder, in this case we are using a custom
        // layout called R.layout.message for each item
        View view = LayoutInflater.from(parent.getContext())
                .inflate(R.layout.message, parent, false);

        return new ChatHolder(view);
    }

    @Override
    protected void onBindViewHolder(ChatHolder holder, int position, Chat model) {
        // Bind the Chat object to the ChatHolder
        // ...
    }
};
```

Finally attach the adapter to your `RecyclerView` with the `RecyclerView#setAdapter()` method.
Don't forget to also set a `LayoutManager`!


#### FirebaseRecyclerAdapter lifecycle

##### Start/stop listening

The `FirebaseRecyclerAdapter` uses an event listener to monitor changes to the Firebase query.
To begin listening for data, call the `startListening()` method. You may want to call this in your
`onStart()` method. Make sure you have finished any authentication necessary to read the data
before calling `startListening()` or your query will fail.

```java
@Override
protected void onStart() {
    super.onStart();
    adapter.startListening();
}
```

Similarly, the `stopListening()` call removes the event  listener and all data in the adapter.
Call this method when the containing Activity or Fragment stops:

```java
@Override
protected void onStop() {
    super.onStop();
    adapter.stopListening();
}
```

##### Automatic listening

If you don't want to manually start/stop listening you can use
[Android Architecture Components][arch-components] to automatically manage the lifecycle of the
`FirebaseRecyclerAdapter`. Pass a `LifecycleOwner` to
`FirebaseRecyclerOptions.Builder#setLifecycleOwner(...)` and FirebaseUI will automatically
start and stop listening in `onStart()` and `onStop()`.

#### Data and error events

When using the `FirebaseRecyclerAdapter` you may want to perform some action every time data
changes or when there is an error. To do this, override the `onDataChanged()` and `onError()`
methods of the adapter:

```java
FirebaseRecyclerAdapter adapter = new FirebaseRecyclerAdapter<Chat, ChatHolder>(options) {
    // ...

    @Override
    public void onDataChanged() {
        // Called each time there is a new data snapshot. You may want to use this method
        // to hide a loading spinner or check for the "no documents" state and update your UI.
        // ...
    }

    @Override
    public void onError(DatabaseError e) {
        // Called when there is an error getting data. You may want to update
        // your UI to display an error message to the user.
        // ...
    }
};
```

### Using the `FirebaseRecyclerPagingAdapter`

The `FirebaseRecyclerPagingAdapter` binds a `Query` to a `RecyclerView` by loading documents in pages.
This results in a time and memory efficient binding, however it gives up the real-time events
afforded by the `FirebaseRecyclerAdapter`.

The `FirebaseRecyclerPagingAdapter` is built on top of the [Android Paging 3 Library][paging-support].
Before using the adapter in your application, you must add a dependency on that library:

```groovy
implementation 'androidx.paging:paging-runtime:3.x.x'
```

First, configure the adapter by building `DatabasePagingOptions`. Since the paging adapter
is not appropriate for a chat application (it would not detect new messages), we will consider
an adapter that loads a generic `Item`:

```java
// The "base query" is a query with no startAt/endAt/limit clauses that the adapter can use
// to form smaller queries for each page.
Query baseQuery = mDatabase.getReference().child("items");

// This configuration comes from the Paging 3 Library
// https://developer.android.com/reference/kotlin/androidx/paging/PagingConfig
PagingConfig config = new PagingConfig(/* page size */ 20, /* prefetchDistance */ 10,
    /* enablePlaceHolders */ false);

// The options for the adapter combine the paging configuration with query information
// and application-specific options for lifecycle, etc.
DatabasePagingOptions<Item> options = new DatabasePagingOptions.Builder<Item>()
        .setLifecycleOwner(this)
        .setQuery(baseQuery, config, Item.class)
        .build();
```

If you need to customize how your model class is parsed, you can use a custom `SnapshotParser`:

```java
...setQuery(..., new SnapshotParser<Item>() {
    @NonNull
    @Override
    public Item parseSnapshot(@NonNull DocumentSnapshot snapshot) {
        return ...;
    }
});
```

Next, create the `FirebaseRecyclerPagingAdapter` object. You should already have a `ViewHolder` subclass
for displaying each item. In this case we will use a custom `ItemViewHolder` class:

```java
FirebaseRecyclerPagingAdapter<Item, ItemViewHolder> adapter =
        new FirebaseRecyclerPagingAdapter<Item, ItemViewHolder>(options) {
            @NonNull
            @Override
            public ItemViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
                // Create the ItemViewHolder
                // ...
            }

            @Override
            protected void onBindViewHolder(@NonNull ItemViewHolder holder,
                                            int position,
                                            @NonNull Item model) {
                // Bind the item to the view holder
                // ...
            }
        };
```

Finally attach the adapter to your `RecyclerView` with the `RecyclerView#setAdapter()` method.
Don't forget to also set a `LayoutManager`!

#### `FirebaseRecyclerPagingAdapter` lifecycle

##### Start/stop listening

The `FirebaseRecyclerPagingAdapter` listens for scrolling events and loads additional pages from the
database only when needed.

To begin populating data, call the `startListening()` method. You may want to call this
in your `onStart()` method. Make sure you have finished any authentication necessary to read the
data before calling `startListening()` or your query will fail.

```java
@Override
protected void onStart() {
    super.onStart();
    adapter.startListening();
}
```

Similarly, the `stopListening()` call freezes the data in the `RecyclerView` and prevents any future
loading of data pages.

Call this method when the containing Activity or Fragment stops:

```java
@Override
protected void onStop() {
    super.onStop();
    adapter.stopListening();
}
```

##### Automatic listening

If you don't want to manually start/stop listening you can use
[Android Architecture Components][arch-components] to automatically manage the lifecycle of the
`FirebaseRecyclerPagingAdapter`. Pass a `LifecycleOwner` to
`DatabasePagingOptions.Builder#setLifecycleOwner(...)` and FirebaseUI will automatically
start and stop listening in `onStart()` and `onStop()`.

#### Paging events

When using the `FirebaseRecyclerPagingAdapter`, you may want to perform some action every time data
changes or when there is an error. To do this:

##### In Java

Use the `addLoadStateListener` method from the adapter:

```java
adapter.addLoadStateListener(new Function1<CombinedLoadStates, Unit>() {
            @Override
            public Unit invoke(CombinedLoadStates states) {
                LoadState refresh = states.getRefresh();
                LoadState append = states.getAppend();

                if (refresh instanceof LoadState.Error || append instanceof LoadState.Error) {
                    // The previous load (either initial or additional) failed. Call
                    // the retry() method in order to retry the load operation.
                    // ...
                }

                if (refresh instanceof LoadState.Loading) {
                    // The initial Load has begun
                    // ...
                }

                if (append instanceof LoadState.Loading) {
                    // The adapter has started to load an additional page
                    // ...
                }

                if (append instanceof LoadState.NotLoading) {
                    LoadState.NotLoading notLoading = (LoadState.NotLoading) append;
                    if (notLoading.getEndOfPaginationReached()) {
                        // The adapter has finished loading all of the data set
                        // ...
                        return null;
                    }

                    if (refresh instanceof LoadState.NotLoading) {
                        // The previous load (either initial or additional) completed
                        // ...
                        return null;
                    }
                }
                return null;
            }
        });
```

#### In Kotlin

Use the `loadStateFlow` exposed by the adapter, in a Coroutine Scope:

```kotlin
// Activities can use lifecycleScope directly, but Fragments should instead use
// viewLifecycleOwner.lifecycleScope.
lifecycleScope.launch {
    pagingAdapter.loadStateFlow.collectLatest { loadStates ->
        when (loadStates.refresh) {
            is LoadState.Error -> {
                // The initial load failed. Call the retry() method
                // in order to retry the load operation.
                // ...
            }
            is LoadState.Loading -> {
                // The initial Load has begun
                // ...
            }
        }

        when (loadStates.append) {
            is LoadState.Error -> {
                // The additional load failed. Call the retry() method
                // in order to retry the load operation.
                // ...
            }
            is LoadState.Loading -> {
                // The adapter has started to load an additional page
                // ...
            }
            is LoadState.NotLoading -> {
                if (loadStates.append.endOfPaginationReached) {
                    // The adapter has finished loading all of the data set
                    // ...
                }
                if (loadStates.refresh is LoadState.NotLoading) {
                    // The previous load (either initial or additional) completed
                    // ...
                }
            }
        }
  }
}
```


## Using FirebaseUI to populate a `ListView`

ListView is the older, yet simpler way to handle lists of items. Using it is analogous to
using a `FirebaseRecyclerAdapter`, but with `FirebaseListAdapter` instead and no `ViewHolder`:

```java
FirebaseListOptions<Chat> options = new FirebaseListOptions.Builder<Chat>()
        .setQuery(query, Chat.class)
        .build();

FirebaseListAdapter<Chat> adapter = new FirebaseListAdapter<Chat>(options) {
    @Override
    protected void populateView(View v, Chat model, int position) {
        // Bind the Chat to the view
        // ...
    }
};
```

## Using FirebaseUI with indexed data

If your data is [properly indexed][indexed-data], change your adapter initialization
to use `setIndexedQuery()`:

```java
// keyQuery - the Firebase location containing the list of keys to be found in dataRef
// dataRef - the Firebase location to watch for data changes. Each key found at
//           keyRef's location represents a list item.
FirebaseRecyclerOptions<Chat> options = new FirebaseRecyclerOptions.Builder<Chat>()
        .setIndexedQuery(keyQuery, dataRef, Chat.class)
        .build();
```

Where `keyQuery` is the location of your keys, and `dataRef` is the location of your data.

### A note on ordering

The order in which you receive your data depends on the order from `keyRef`, not `dataRef`:

```javascript
{
  "data": {
    // This order doesn't matter, the order is taken from keys/(user1 or user2).
    "3": true,
    "1": "some data",
    "2": 5
  },
  "keys": {
    // These two users have different orders for their data thanks to key side ordering.
    "user1": {
      "1": true,
      "2": true,
      "3": true
    },
    "user2": {
      "3": true,
      "2": true,
      "1": true
    }
  }
}
```

[firebase-lists]: https://firebase.google.com/docs/database/android/lists-of-data
[indexed-data]: https://firebase.google.com/docs/database/android/structure-data#best_practices_for_data_structure
[recyclerview]: https://developer.android.com/reference/androidx/recyclerview/widget/RecyclerView
[arch-components]: https://developer.android.com/topic/libraries/architecture/index.html
[paging-support]: https://developer.android.com/topic/libraries/architecture/paging/v3-overview
