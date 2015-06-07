---
layout: post
title:  "Accessing Meteor Package Data"
date:   2015-08-20 12:06:57
categories: meteor
comments: true
---

## Motivation

In mid-December, a new page appeared on the [Atmosphere](http://atmospherejs.com) website with a [FAQ](https://atmospherejs.com/i/faq), answering questions which had presumably been asked an enormous number of times by the Meteor community.  Amongst the answers was the following:

 > For most packaging API needs it would make more sense to go direct to the source (Meteor’s packaging server at packages.meteor.com). The API to do so is not yet documented, but if enough people need it they (the Meteor Development Group) would probably do so.

It turns out that pulling data from [packages.meteor.com](http://packages.meteor.com) from within a Meteor app is actually pretty straightforward, so I thought I'd document it.  Please note that this is all the result of a few hours investigation, so there could well be errors or ommissions, for which I welcome corrections.

## Local Catalogue

Whilst I don't profess to having a complete understanding of the tools that ship with Meteor, an exploration of [the code](https://github.com/meteor/meteor/tree/bd54f09e4ce299035c2ad57e02f558d64f6b0a93/tools) makes it fairly clear that the local catalogue which a user accesses from the command-line is an SQLite database which is updated when various actions are performed.

The mechanism via which the update takes place is a Meteor [method call](http://docs.meteor.com/#/full/meteor_call) named 'syncNewPackageData' on line 83 of [package-client.js](https://github.com/meteor/meteor/blob/bd54f09e4ce299035c2ad57e02f558d64f6b0a93/tools/package-client.js), the arguments to which can largely be inferred from the comments in the surrounding functions.

## Connecting to the package server from your own app

The first thing you need to do in order to build your own package catalogue (apologies for the UK spelling) within a Meteor app is to set up a [remote DDP connection](http://docs.meteor.com/#/full/ddp_connect) to the package server at [packages.meteor.com](http://packages.meteor.com).  This sort of thing is [well documented elsewhere](http://stackoverflow.com/questions/18358526/connect-two-meteor-applications-using-ddp?rq=1), so suffice to say that this should do the trick:

{% highlight javascript %}
remote = DDP.connect('http://packages.meteor.com');
{% endhighlight %}

**NOTE** - you'll need to do this from the server as it won't work via AJAX due to a lack of CORS headers.

## Calling `syncNewPackageData`

You should now be able to use `remote` to call `syncNewPackageData` from your own Meteor server almost like calling a Meteor.method within your own app.

 {% highlight javascript %}
remote.call('syncNewPackageData', syncToken, syncOpts, callback);
{% endhighlight %}

Assuming success, the result looks like so:

![syncPackage object](/assets/syncPackage.png)

* The `collections` property contains up to 500 entries in the listed categories.  The ones which will be of interest to package enthusiasts will be `packages` and `versions` (which actually contains far more metadata):

![syncPackage collections](/assets/syncPackageCollections.png)

It also contains some other useful properties:

`syncToken` - this contains a series of UNIX dates which indicate the dateTime up to which results have been returned.  It can be passed back into the `syncNewPackageData` invocation to receive another batch of results, which is useful in two different circumstances:

1. Easy paging if you require the full catalogue to be returned.  Simply pass an empty object as the syncToken on the first invocation, and then the last received syncToken on subsequent invocations until all results have been received (there's an example of this below).
2. Easy updates when all you require is a diff.  Pass the most recently received syncToken, and you'll just get the packages/versions released since then.

`upToDate` - this is true if the returned data includes the most recent for all collections; use it to determine when you can stop requesting further pages.

I'll leave you to peruse the [source](https://github.com/meteor/meteor/blob/bd54f09e4ce299035c2ad57e02f558d64f6b0a93/tools/package-client.js) for details on syncOpts, which doesn't appear to be especially useful.

## Example - downloading the entire package catalogue

 {% highlight javascript %}
var Future = Npm.require('fibers/future');

function getPackages() {

  var fut = new Future(),
    syncToken = {},
    collections = {},
    count = 1,
    packageRequest = function(cb) {
      remote.call('syncNewPackageData', syncToken, {}, function(err, res) {
        console.log('Page ', count++);
        if (err) fut.throw(new Meteor.Error(err));
        if (!res) fut.throw(new Meteor.Error('no_results', 'No results returned'));
        syncToken = res.syncToken || {};
        _.each(res.collections, function(val, key) {
          if (_.has(collections, key))
            collections[key] = collections[key].concat(val);
          else
            collections[key] = val;
        });
        // Using setImmediate to allow GC to run each time in case there are a LOT of pages
        if (!res.upToDate) setImmediate(Meteor.bindEnvironment(packageRequest.bind(this, cb)));
        else cb();
      });
    }

  packageRequest(function() {
    fut.return({
      collections: collections,
      syncToken: syncToken
    });
  });

  return fut.wait();

}
{% endhighlight %}

## Limitations

The chief limitation of this approach is that potentially useful metadata is missing: namely, the number of downloads.  Whilst the number of stars is probable only stored in Atmosphere's DB, logic dictates that downloads must be aggregated by Troposphere (packages.meteor.com) since that's the endpoint the user accesses from the command-line when they're added.  If anybody has any ideas how this data can be accessed (if at all), I'd love to hear them.

Atmosphere metadata can also be pulled from their server via DDP (as demonstrated by [Vianney Lecroart's Fastosphere project](https://github.com/acemtp/meteor-fastosphere)), but given that this involves subscribing rather than method calling, I think some caution is required with this sort of remote connection to avoid placing excess load on the remote server, so I would advise studying his code if this interests you.  Vianney polls every 12 hours to take a complete snapshot of the package data and stores it in a dedicated index, thereby limiting the impact on the remote server to a bare minimum, rather than maintaining an open subscription and reactively receiving updates.
