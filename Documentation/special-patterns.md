# Special Patterns

## Messenger

When observer and observable need to be more decoupled, it is common to use a mediating observable through which any object can anonymously send updates. An example of this mediator is `Foundation`'s `NotificationCenter`.

This extension of the *Observer Pattern* is sometimes called *Messenger*, *Notifier*, *Dispatcher*, *Event Emitter* or *Decoupler*. Its main differences to direct observation are:

- An observer may indirectly observe multiple other objects.
- Observers don't care who triggered an update.
- Observer types don't need to depend on the types that trigger updates.
- Updates function more as messages (notifications, events) than as artifacts of raw data.
- Every object can trigger updates, without adopting any protocol.
- Multiple objects may share the same update type and trigger the same updates.

You can simply use a global (mapped) `Variable` as a mediating messenger:

~~~swift
let textMessenger = Var<String>().new()
observer.observe(textMessenger)
{
    textMessage in
    
    // respond to text message
}
    
textMessenger.send("some message")
~~~
    
An `Observer` can use the select filter to observe one specific message:

~~~swift
observer.observe(textMessenger, select: "event name")
{
    // respond to "event name"
}
~~~
    
Of course, if you'd wanna acces the latest message, just backup the messenger with a variable:

~~~swift
let currentMessage = Var<String>()
let textMessenger = currentMessage.new()
~~~

## Nested Messenger

A *Nested Messenger* is a helpful, and sometimes necessary, application of the *Messenger* pattern.

Instead of making a class `C` directly observable you give it an observable messenger as a property. `C` sends its updates via its messenger, and observers of `C` actually observe the messenger of `C`:

~~~swift
class C {
   let messenger = Messenger()
   
   class Messenger: Observable {
      latesUpdate = Event.didNothing
      enum Event { case didNothing }
   }
}
~~~

An why would you want that? A *Nested Messenger* is necessary in three scenarios ...

### 1. Require Specific Observability in an Interface

We want to declare a variable or constant as conforming to an interface (let's say `Database`) specifying (among other functionality) observability with a specific update type (say `DatabaseUpdate`).

#### Challenge

We don't want to define an abstract base class because objects conforming to the interface should be able to derive from their own (and more meaningful) class (like `ICloudDatabase`).

Now, we would want to define a protocol like this:

~~~swift
protocol Database: Observable where UpdateType == DatabaseUpdate { }
~~~

But this protocol could only be used as a generic constraint because it has an associated type requirement (Swift doesn't have generalized existentials yet).

We can't declare a variable or constant of the protocol type `Database`, like we are used to with delegate protocols:

~~~swift
weak var delegate: MyDelegateProtocol
// ^^ perfectly fine

var database: Database
// ^^ compiler error: Protocol 'Database' can only be used as a generic constraint because it has Self or associated type requirements
~~~

#### Solution

We use a `Database` protocol but without a `where` clause. Instead, we define a `DatabaseMessenger` similar to *Nested Messengers* and require the `Database` to have such a messenger: 

~~~swift
protocol Database {
   var messenger: DatabaseMessenger { get }
   // declare other functionality
}

class DatabaseMessenger: Observable {
   // declare update type etc.
}
~~~

Now, we must route all observation of the database through its messenger, but at least it works.

### 2. Observe Apple Classes that Can't be Referenced Weakly

There are a number of classes from Apple's frameworks that [cannot be referenced weakly](https://developer.apple.com/library/archive/releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html#//apple_ref/doc/uid/TP40011226-CH1-SW17). Among them are `NSMenuView`, `NSFont` and `NSTextView`.

When we create a custom `NSTextView` and try to observe it, we get a runtime error:

~~~swift
class MyTextView: NSTextView: Observable {
   // declare update type etc.
}

let textView = MyTextView()

observe(textView) { update in
   // process update
}

// the error reads:
// objc[89748]: Cannot form weak reference to instance (0x600000c8a5e0) of class NSTextView. It is possible that this object was over-released, or is in the process of deallocation.
~~~

So, once again, we use a nested messenger:

~~~swift
class MyTextView: NSTextView {
   let messenger: Messenger
   
   class Messenger: Observable {
      // declare update type etc.
   }
}

let textView = MyTextView()

observe(textView.messenger) { update in
   // process update
}
~~~

### 3. Inherit and Extend Observability

Consider this case: I have a generic class `Tree`. It is `Observable`, so tree nodes can observe their branches. Then I have an `Item` which derives from `Tree`. `Item` cannot extend or override the `Tree.UpdateType`.

In order to further specify what items can send to their observers, the `Tree` must use a nested messenger. This tree messenger should (somewhat redundantly) be named after its class: `treeMessenger`, so that there's no confusion in inheriting classes about which messenger belongs to which ancestor.