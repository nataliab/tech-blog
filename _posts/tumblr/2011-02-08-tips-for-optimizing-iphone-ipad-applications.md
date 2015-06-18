---
layout: post
title: Tips for Optimizing iPhone/iPad Applications (Video)
date: '2011-02-08T17:22:00-05:00'
tags:
- iPhone
- iPad
- iOS
- gilt
- gilttech
- mobile development
- video
- optimization
- NSOperationQueue
- "@synchronized"
- NSDateFormatter
- iOS app performance
- Evan Coyne Maloney
- submission
tumblr_url: http://tech.gilt.com/post/3187131303/tips-for-optimizing-iphone-ipad-applications
---
We’ve made a number of optimizations to our iPhone/iPad codebase over the past couple of months, the effects of which can be seen in the attached video:



(Because our application is so dependent on the network, the new code won’t appear to outperform the old code 100% of the time. This is due to the vagaries of the connection between the devices and the server.)

As we prepare to submit this latest code to Apple, we decided to make some of our optimization techniques public in the hopes that it may benefit developers outside of our company:

In a number of places in our code, we were spawning new threads to perform short-lived operations. We’ve rewritten a number of these as NSOperations, allowing us to use NSOperationQueue instances, which are implemented under the hood by the OS as thread pools. So we’re spawning fewer new threads and reusing existing threads more; as a result, we’ve avoided some overhead of launching new threads. Also, since the OS is able to manage threading behavior for us via the NSOperationQueue, the OS is able to make smart decisions at runtime about thread resource usage that we were previously hard-coding at development time. Our assumptions could never be correct for all cases, since each device may have a different number of other processes running at any time. The OS itself is in the best position to make such decisions.
Using the @synchronized construct for thread safety in iOS is very expensive. (Part of the reason for this is that @synchronized sets up inherent exception handlers.) So, one major optimization was to replace @synchronized blocks with using the NSLock/NSRecursiveLock classes for thread safety. Doing this reduced thread waits that occur during lock contention, improving app performance overall.
Another way we avoid unnecessary locking is by using the thread-local storage provided by the threadDictionary: method of NSThread. Thread-unsafe resources that are expensive to construct—but that don’t take up a lot of memory—are good candidates for this. So, instead of constructing many short-lived objects on an as-needed basis, we can construct one per thread and keep it in thread-local storage. This ensures that the resources are used in a thread-safe manner, while avoiding the expense of repeated instantiations. (This technique is less useful for short-lived threads that are spawned often, because in those cases you won’t be avoiding the instantiation costs. Note, too, that Apple recommends against using thread-local storage for threads managed by NSOperationQueues.)
We discovered that instantiating and using the NSDateFormatter class is particularly expensive. (And, because the NSDateFormatter class is not thread-safe, we can’t share instances across threads.) We use this class to parse text dates that arrive from the server. So, to avoid the instantiation expense, we’re caching NSDateFormatters in thread-local storage. Also, in our particular case, because so many of the date strings we’re parsing turn out to be identical (a result of many sales starting at the same time), we’re avoiding some of the date parsing altogether by maintaining a cache with date string keys that point to the NSDate instances that resulted from parsing that date string. Most of our dates now don’t need to be parsed at all; we just need to look up the NSDate associated with a given date string. Doing this shaved 5%-10% off the processing done at app startup.
Our old caching system used a SQLite database; we’re now using the filesystem for caching, which avoids all the marshalling/unmarshalling required by the database accesses. This is particularly beneficial for our image cache. Now that cached images are stored in the filesystem, we can load them directly with [UIImage imageWithContentsOfFile:]. Previously, our image data was first loaded into an NSData instance that was then passed to [UIImage imageWithData:]. Not only does this new technique perform better, but by using this method to reference images, the OS has the option of purging image data from memory and reconstituting it directly from the file on an as-needed basis. (Whether or not a given version of iOS does this is another story, but at least this technique gives the OS that option.) That’s not possible if we were still using [UIImage imageWithData:]; the OS can’t purge the image data in that case since there would be no way to transparently reconstitute it if the image data were needed again. This gives the OS the ability to better manage memory on our behalf.
Since a big factor in iOS app performance is the memory footprint of the application, we’re also making better decisions about what images to store in our memory cache. Large images in iOS are particularly expensive, since the operating system stores image data uncompressed in memory. The large Gilt sale images are over 6MB uncompressed, and we’re now avoiding caching those in memory altogether. (We now only cache images in memory if their uncompressed size is less than 1MB.) This reduces the amount of memory warnings our app has to handle, which means we don’t have to flush our memory caches as often. As a result, all of our various memory caches have more cache hits and fewer misses; our memory caches are now more efficient.
If there’s any interest in my going into further detail on any of these items, let me know. In the meantime, I hope you find this helpful!

Evan Coyne Maloney Senior Software Engineer, Mobile Team Gilt Groupe