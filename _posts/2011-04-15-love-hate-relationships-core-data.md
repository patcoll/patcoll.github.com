---
layout: post
title: "Love and Hate Relationships: Core Data"
---

{{ page.title }}
================

<p class="meta">15 April 2011 - Pittsburgh</p>

I have been working with a lot of Objective-C and Cocoa for iOS lately.

The joys and sorrows of Cocoa have evened themselves out, mostly.

Developing for a popular mobile platform like iOS has its advantages. A lot of attention is paid to the entire infrastructure and ecosystem, which makes developing for it an easier task. Xcode is a great tool. There are data structures that [optimize themselves][2]. There are well-designed components for handling a lot of common tasks. StackOverflow has an endless supply of answered newbie questions that help enormously<sup class="reference">1</sup>.

There is one unique set of sorrow and joy I'd like to share today, and it has to do with the data storage layer in Cocoa known as Core Data.

## What happened

As I came to find out, managed objects (objects that inherit from `NSManagedObject`) are always contained within a managed object context (`NSManagedObjectContext`) in an application that uses Core Data. Managed object contexts (MOCs) are like scratchpads, in that you can create, modify or deleted managed objects in a context, but those changes do not actually get persisted to the data store unless the context is explicitly saved.

(Design patterns FTW on this one: I had worked with this pattern before in well-designed ORMs like [Doctrine 2][3] and [SQLAlchemy][4] so it was easy to adapt to. I prefer to call it a "context" now. The name really drives home the meaning.)

So my main user interface (UI) was constantly saving changes to objects in the foreground then persisting these changes to Core Data and ultimately to a web service. Every so often, a `sync` action would download new information from the web service and change the data accordingly in Core Data. This all worked fine.

## Sorrow

When a new requirement came down the pike I had to change the design to handle more objects in bulk. No biggie, right? Wrong. The more objects that the app had to handle at once, the more the `sync` froze the UI by saving its changes. To make matters worse -- I was getting unexplained data conflicts and crashes because these data changes were seemingly pulling the rug from underneath what was happening in both the background and foreground. What was happening?

When I dug deeper, I discovered that even though the `sync` appeared like it was performing in the background, it was really performing its work on the main thread, which froze the UI. This wasn't really noticeable when I was dealing with one object at a time, but bulk processing these objects was a different story. I needed to change my approach.

Also, because both the "background" and the "foreground" processes were happening on the main thread, and overlapping a little, they both were trying to change the same objects, causing conflicts and crashes that were almost impossible to reproduce in a consistent manner.

## Joy

I realized I needed a real solution for processing the same data set in multiple places. Enter threads.

You know those well-designed components I mentioned? They helped save my sanity. Through reading some really well-written articles<sup class="reference">2</sup> I discovered that the key was to maintain separate managed object contexts for each set of data manipulation you do. Syncing in the background? A new MOC. Editing in the foreground? A new MOC.

I ended up implementing the approach that describes how to [maintain and synchronize data among thread-specific MOCs][8]. That, combined with running the `sync` process in a true background thread, helped alleviate my Core Data woes. I'll include some obfuscated code here for completeness, and hopefully clarity.

<script src="https://gist.github.com/922466.js"> </script>





<small>

1. As an aside, being told to RTFM as a beginner in a particular language or framework is really frustrating. As a programmer I have a good base of knowledge I can use to carefully craft my Google searches. I am eternally grateful for non-RTFM folks who have an ounce of patience to craft their kind answers (or debates) just as carefully, if not more so. The programming community, especially the new guns who learn everything through self-discovery and Google, would be nowhere without these people. Myself included.

2. Highly-recommended Core Data articles
    * [Concurrency with Core Data][5]
    * [Using Core Data on Multiple Threads][6]
    * [Multiple Managed Object Contexts with Core Data][7]
    * [NSManagedObjectContextDidSaveNotification across MOCs on two threads][8]

</small>

[2]:http://ridiculousfish.com/blog/archives/2005/12/23/array/

[3]:http://www.doctrine-project.org/docs/orm/2.0/en/tutorials/getting-started-xml-edition.html#obtaining-the-entitymanager

[4]:http://www.sqlalchemy.org/docs/orm/tutorial.html#creating-a-session

[5]:http://developer.apple.com/library/ios/#documentation/cocoa/conceptual/CoreData/Articles/cdConcurrency.html

[6]:http://www.duckrowing.com/2010/03/11/using-core-data-on-multiple-threads/

[7]:http://www.timisted.net/blog/archive/multiple-managed-object-contexts-with-core-data/

[8]:http://www.cocoabuilder.com/archive/cocoa/293250-nsmanagedobjectcontextdidsavenotification-across-mocs-on-two-threads.html#293295


<div id="disqus_thread"></div>
<script type="text/javascript">
    /* * * CONFIGURATION VARIABLES: EDIT BEFORE PASTING INTO YOUR WEBPAGE * * */
    var disqus_shortname = 'patcoll'; // required: replace example with your forum shortname

    // The following are highly recommended additional parameters. Remove the slashes in front to use.
    var disqus_identifier = '2011-04-15-love-hate-relationships-core-data';
    var disqus_url = 'http://www.patcoll.com/2011/04/15/love-hate-relationships-core-data.html';

    /* * * DON'T EDIT BELOW THIS LINE * * */
    (function() {
        var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
        dsq.src = 'http://' + disqus_shortname + '.disqus.com/embed.js';
        (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
    })();
</script>
<noscript>Please enable JavaScript to view the <a href="http://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
<a href="http://disqus.com" class="dsq-brlink">blog comments powered by <span class="logo-disqus">Disqus</span></a>
