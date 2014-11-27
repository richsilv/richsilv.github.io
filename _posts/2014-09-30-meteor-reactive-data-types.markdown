---
layout: post
title:  "Reactive Data in Meteor"
date:   2014-09-30 10:58:45
categories: meteor
comments: true
---

## Motivation

Much of the perceived magic that Meteor provides is the result of its wonderful reactive data model, which, when implemented properly, sees your app update itself automatically on changes in state or data, without the need for any DOM manipulation on the part of the developer.  However, working out exactly *what* is reactive, *how* it's reactive and *when* to use it is something that may not be completely obvious for those at the start of their journey with Meteor.

# Reactivity out of the box

A cursory investigation of the [ever-instructive docs](http://docs.meteor.com/#reactivity) provides the following:

> ...the reactive data sources that can trigger changes are: 
>
>* Session variables
>* Database queries on Collections
>* Meteor.status
>* The ready() method on a subscription handle
>* Meteor.user
>* Meteor.userId
>* Meteor.loggingIn

As useful as the latter five items in that list may be, they're clearly only appropriate for very specific circumstances, which leaves only the first two available for more generalised reactivity.

We'll have a look at the benefits of both, as well as observing that there are actually other ways of achieving reactivity, both using MDG packages and by building reactive sources oneself.

## Session variables

###### The documentation can be [read here](http://docs.meteor.com/#session).

*Session* provides a convenient API for storing and retrieving arbitrary key-value pairs, which are *reactive* by default.  What this means is that template helpers or `Tracker.autorun` blocks will automatically rerun if they contain a `Session.get` method call with a key which has been `Session.set` elsewhere.  For example:

{% highlight javascript %}
Session.setDefault('myName', 'Richard');

Template.greeting.helpers({
	myName: function() {
		return Session.get('myName');
	}
});
{% endhighlight %}

{% highlight html %}
<template name="greeting">
	<p>Hello there, {% raw %}{{myName}}{% endraw %}</p>
</template>
{% endhighlight %}

At this point, if we put `Session.set('myName', 'Claire')` into the browser console, the `<p>` tag in the `greeting` template will update with the new name, as we would expect with Meteor reactivity.

Here are the key things I think you should bear in mind if you're intending to use Session variables:

1. Session values persist across hot reloads.  This is potentially useful during development, but it's something you need to be aware of to avoid unexpected behaviour.
2. The API isn't particularly extensive.  In addition to `get` and `set`, you're provided with `equals` and `setDefault`, but this is as far as Session methods extend, which can seem quite clunky if you're storing large, nested objects this way.  Of course, you could write your own helper methods, but there may be easier alternatives (see below).
3. The values you store will always sit in the global namespace.  Whilst this shouldn't be a problem from the perspective of namespace conflicts (as they're all wrapped inside the `Session` object), you need to be aware that a Meteor-savvy user can always `get` and `set` any Session value in the browser console, whether you'd rather it be private or not.
4. Session variables **will not** cause their dependencies to rerun when `Session.set` is supplied with a value which is equal to the existing value.  This may be an advantage as it potentially reduces DOM thrashing, but bear in mind that you cannot assume dependencies will rerun simply because you've called `Session.set`.

## Database queries on Collections

###### The documentation can be [read here](http://docs.meteor.com/#collections).

The topic of Collections and their use in Meteor applications is a very large one indeed, so I'll keep this section brief.  However, these are the main points regarding reactive data:

1. It's perfectly possible (and often very useful) to define *client-only* collections, which will be sessional, and won't be synchronised with the server.  These give you the full power of the minimongo API without any pub/sub overhead, and have reactivity baked in.
2. For collections to drive reactivity, you actually need to be querying the database from within the Template helper or autorun block; i.e. there needs to be a `Collection.find` or `findOne` or `count` within the helper or autorun - you can't `fetch()` results somewhere else, store them in a variable and expect references to that variable to be reactive (unless of course it's a reactive variable...).
3. You can also define a *private, client-only* collection with the `var` keyword (i.e. `var MyCollection = new Mongo.Collection(null)`).  This will provide a private, reactive structure, which can be used as a key/value store alternative to Session variables.  The only difficulty is that you will only be able to update values by using the `_id` field (which you'll either have to repeatedly query or else store somewhere) due to the way Meteor treats client-side code.  Like Session variables, you can also only store [EJSON](http://docs.meteor.com/#ejson)-able values.

# ReactiveVar and ReactiveDict Packages

In fact, Meteor also ships with two other reactive data sources out of the box, **ReactiveVar** and **ReactiveDict**.  Both need to be added as packages (even though they're present in the standard distribution), and only ReactiveVar appears in the official documentation, but both can be extremely useful for developers.

## ReactiveVar

###### The documentation can be [read here](http://docs.meteor.com/#reactivevar_pkg).

To add the ReactiveVar constructor to your project:

{% highlight bash %}
$ meteor add reactive-var
{% endhighlight %}

ReactiveVar is in some respects an atomic reactive unit, like a single Session variable, with the disadvantage that its values will not persist across hot code pushes.  However, there are lots of advantages:

1. ReactiveVar instances can be scoped as any normal Javascript variable, and can contain any value - not just EJSON.
2. They also allow you (although they don't compel you) to define a proprietary `equals` function, which will determine the exact circumstances under which resetting the value of the variable will invalidate dependent computations.  This allows you to very easily overwrite the default, Session-like behaviour if required, and ensure that the setting of a ReactiveVar value will always invalidate dependent computations. You could also supply even more subtle logic.
3. However, by default, updating the value of a ReactiveVar will not invalidate computations which depend on it *if the new value is the same as the existing one*, exactly like Session variables (provided no `equals` function is supplied).
4. To reiterate, unlike Session variables, the value of a ReactiveVar will not be retained across hot code pushes.

## ReactiveDict

###### The (rather limited) documentation appears in [the code on Github](https://github.com/meteor/meteor/blob/devel/packages/reactive-dict/reactive-dict.js).

To add the ReactiveDict constructor to your project:

{% highlight bash %}
$ meteor add reactive-dict
{% endhighlight %}

ReactiveDict is the prototype which is used to construct the global `Session` object.  What this means is that all the familiar Session methods and properties are available for any other ReactiveDict you might construct, except you wouldn't necessarily have to leave the parent object in the global namespace.  Some other things to bear in mind:

1. You can pass an object containing migration data into the constructor - i.e. you can seed the ReactiveDict when it's created: `myDict = new ReactiveDict({foo: 'bar'});`
2. Given that, like a Session object, each key has its own associated dependency, the only reason for creating multiple ReactiveDicts is to keep the information in each private.  Beyond this, segregating your keys into different ReactiveDicts will have no impact on your app's reactivity.

# Roll Your Own

As useful as the provided reactive data types described above can be in getting the most out of Meteor, there are times when building your own is the right solution.  The good news is that this is far easier than you might imagine.

## Tracker.Dependency: the building block of reactivity

The basic building block of reactivity is the `Tracker.Dependency` prototype.  An object thus constructed has two simple but powerful methods:

* `depend` - this instructs the computation from which this method is called to rerun when the associated dependency registers a `changed` event.
* `changed` - this fires the change (thereby invalidating all the `depend`ent computations).

The best way to illustrate the use of `Tracker.Dependency` is with a simple example.

## Proprietary Reactivity 101: the simplest example I can think of

{% highlight javascript %}
myDep = new Tracker.Dependency();

Tracker.autorun(function(comp) {
	myDep.depend();
	console.log("Something has happened!");
});
{% endhighlight %}

If you add the code above to your Meteor javascript, you can then enter `myDep.changed()` in the console, and will be rewarded with the expected message.  This is as simple as dependencies get: no explicit data dependency at all, just an object which, when touched, causes all dependent computations to rerun.  Experimenting with this code is a great way to understand how these dependencies work, where they can be applied (try putting the above in a template helper), as well as the rules that govern `autorun` blocks (although that's a whole [topic in itself](http://docs.meteor.com/#tracker_autorun)).

## Constructing a proper reactive variable

There are undoubtedly some cases for which the simple example above could prove useful, but in general we are going to require some sort of data to be associated with our dependency.  To solve this, we can roll our own equivalent of a ReactiveVar:

{% highlight javascript %}
MyDep = function(initial) {
  this.value = initial;
  this.dep = new Tracker.Dependency();
};

MyDep.prototype.get = function() {
    this.dep.depend();
    return this.value;
};

MyDep.prototype.set = function(newValue){
    if (this.value !== newValue) {
        this.value = newValue;
        this.dep.changed();
    }
    return this.value;
};
{% endhighlight %}

The example above should be relatively self-explanatory, and you can see how a call to `MyDep.get` will in effect register a `depend` call within the enclosing computation, hooking up reactivity, and subsequent `MyDep.set` calls will spark invalidation via the `changed` call (provided the old value differs from the current one).

## Taking things further

At this point there are a huge number of other things you could do to customise the reactive data source you've just built.  I'll leave them for you to investigate, but here are some ideas:

* Removing the requirement that the new value differs from the old one, so that dependent computations are always invalidated.
* Returning cloned results from the `get` method; hopefully you will realise that if the `value` of a `MyDep` is an object, the `get` method will return *a reference to that object* rather than a copy of it (and if you don't, read some Crockford!).  This can cause unexpected behaviour, and it may be better to explicitly test for this and clone an object value before returning it, depending on your use case.
* Adding the facility to `get` and `set` individual keys within your object, possibly recursively.  Rather than having a whole array of individual dependencies relating to the different keys (like we saw in ReactiveDict), maybe it would be better to have a single dependency, which is always `changed` whenever one of the keys is changed, but has an API that means you don't have to `get` the whole object just to get the value of one of its keys.
* Adding some sort of debug counter to keep track of the total number of invalidations your reactive variable is registering.  You could have a development mode which stopped registering `changed` calls beyond a certain number to stop your browser hanging and allow you to do some debugging.  This is immensely helpful if you ever have problems with infinite invalidation loops!

Some of the ideas above are implemented in this [MeteorPad](http://meteorpad.com/pad/asdkFDnpZpS7KZQT7).  Let me know if you can think of any more!
