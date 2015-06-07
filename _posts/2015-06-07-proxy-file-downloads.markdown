---
layout: post
title:  "Proxy File Downloads in Meteor"
date:   2015-06-07 21:44:31
categories: meteor
comments: true
---

## Motivation

I haven't blogged about Meteor in a long time (in 2015, in fact), so it's about time.
Conveniently, a recent Meteor project which required proxy routes to file downloads provides a great opportunity, as it's a feature that may well be of interest to others and is surprisingly easy to set up.

## Under the hood

The Meteor core package [webapp](https://github.com/meteor/meteor/tree/devel/packages/webapp) uses [connect](https://www.npmjs.com/package/connect) under the hood to serve content to browsers, and it's relatively easy to use [`WebApp.connectHandlers`](https://docs.meteor.com/#/full/webapp) to register server-side routes from which to serve content outside your main web app.

However, extra functionality can be provided by adding the (simple:json-routes)[https://github.com/stubailo/meteor-rest/tree/master/packages/json-routes] package by core dev Sashko; ostensibly, its purpose is to allow apps to respond to requests (optionally including json bodies) to respond with json objects.  None of that's required here, but the package also includes (connect-route)[https://github.com/baryshev/connect-route], a router for Connect which allows parameters to be passed in the URL.

{% highlight shell %}
meteor add simple:json-routes
{% endhighlight %}

## A Contrived Example

Let's say we have three individuals in a collection, all of which have associated files on the local filesystem.  We'd like to set up a route from which we can download the relevant file by supplying the individual's name.

{% highlight javascript %}
People = new Mongo.Collection('people');

if (Meteor.isServer) {

  Meteor.startup(function () {

    if (!People.findOne()) {
      People.insert({name: 'alice', file: 'file-a.dat'});
      People.insert({name: 'bob', file: 'file-b.dat'});
      People.insert({name: 'charlie', file: 'file-a.dat'});
    }

  });

}
{% endhighlight %}

## Registering Server-side Routes

The (JSON-Routes API)[https://github.com/stubailo/meteor-rest/tree/master/packages/json-routes] now makes it trivially easy to register an appropriate route, which we can use to return the file in question as a download.

{% highlight javascript %}
// THIS CODE SHOULD BE RUN ONLY ON THE SERVER

// we can Npm.require fs as a core Node package, but we'd need to add
// meteorhacks:npm if we wanted to get the file stream from elsewhere
// (for example aws-sdk)
var fs = Npm.require('fs');

JsonRoutes.add('get', '/file/:name', function(req, res, next) {

  var person = People.findOne({name: req.params.name});

  if (person) {

    // indicate a download and set the filename of the returned file
    res.writeHead(200, {
      'Content-Disposition': 'attachment; filename=' + person.file,
    });
    // read a stream from the local filesystem, and pipe it to the response object
    // note that anything you put in the `private` directory will sit in
    // assets/app/ when the application has been built
    fs.createReadStream('assets/app/' + person.file).pipe(res);

  } else {

    // otherwise indicate that the name is not recognised
    res.writeHead(400);
    res.end('cannot find ' + req.params.name);

  }

});
{% endhighlight %}

Two things to point out from the above example:

1. There's no error-handling above, so if the app is unable to find the file in question it will simply throw.
2. A more realistic example probably wouldn't be reading files from the local filesystem, but more likely from dedicated cloud storage.  However, it's trivial to adapt this code to serve files by creating readable streams from (AWS S3)[http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#getObject-property] or (Google Cloud Storage)[https://googlecloudplatform.github.io/gcloud-node/#/docs/v0.14.0/storage/file?method=createReadStream].  This way, the objects in question can remain both private and hidden, and only accessible to the public via your Meteor API.

Comments appreciated.
