---
layout: post
title:  "Scheduling events in the future with Meteor (Email example)"
date:   2014-12-12 09:46:14
categories: meteor
comments: true
---

## Motivation

Having just implemented a server-side task scheduler with Meteor that allows the user to schedule a given task to take place exactly once in the future, and then answered a question on the same subject on [Stack Overflow](http://stackoverflow.com/questions/27430689/how-to-get-email-send-to-send-emails-in-the-future-7-days-14-days-from-now-et/27440807#27440807), I thought I'd share my method here.

## Synced-Cron

As the number of Meteor packages on Atmosphere increases exponentially, one of the best guarantees of quality is the `percolatestudio` namespace.  They've released the excellent `synced-cron` package, which is far more powerful than the requirements of this use case, but it still works better than any other package I've come across.

```
$ meteor add percolatestudio:synced-cron
```

## Set up your task and a schedule collection

Set up a server-side collection to store your tasks in case of server reboot, and the body of the task you want to complete (in this case, sending an email).

```javascript

    FutureTasks = new Meteor.Collection('future_tasks'); // server-side only

	// In this case, "details" should be an object containing a date, plus required e-mail details (recipient, content, etc.)

	function sendMail(details) {

		Email.send({
			from: details.from,
            to: details.to,
            etc....
        });

	}
```

## Add functions to schedule and record your tasks

```javascript

    function addTask(id, details) {

	    SyncedCron.add({
			name: id,
			schedule: function(parser) {
				return parser.recur().on(details.date).fullDate();
			},
			job: function() {
				sendMail(details);
	            FutureTasks.remove(id);
				SyncedCron.remove(id);
	            return id;
			}
		});

    }

    function scheduleEmail(details) { 

		if (details.date < new Date()) {
			sendMail(details);
		} else {
		    var thisId = FutureTasks.insert(details);
		    addTask(thisId, details);		
		}
		return true;

	}
```

## Process existing tasks on reboot, and start the Cron

```javascript

	Meteor.startup(function() {

		FutureTasks.find().forEach(function(mail) {
			if (mail.date < new Date()) {
				sendMail(mail)
			} else {
				addCronMail(mail._id, mail);
			}
		});
		SyncedCron.start();

	});
```

## Summary

Hopefully, the code above is straightforward, as is the path to generalising to other use cases beyond sending email.  If there are any questions or suggestions, just let me know.
