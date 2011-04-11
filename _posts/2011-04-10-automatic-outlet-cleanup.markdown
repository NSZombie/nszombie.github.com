---
author: Jason Foreman
title: Automatic cleanup of IBOutlets
layout: post-nc
tags: ['iOS', 'UIKit']
---

One of the more common sources of confusion among iOS developers is memory management.  Even after mastering the basic [memory management rules][THE RULES!], many developers fail to properly manage memory related to IBOutlets and objects loaded from a nib across the entire view controller lifetime.  In this article we examine that problem in detail and present a solution to make IBOutlet management much more simple.

## Understanding IBOutlet lifetime

[Apple's documentation recommends][nib mm] using a nonatomic, retained property for implementing IBOutlets on iOS. This means that every object that is loaded from a nib an assigned to such a property will remain in memory until is is released by the view controller, typically by setting the property to `nil`. By following the normal memory management rules, a developer can ensure that any retained objects are released in a properly implemented `-dealloc` method. But the story gets more interesting when you take a closer look at view controller lifetime and the often overlooked or misunderstood `-viewDidUnload` method.

### Fucking -viewDidUnload, how does it work?

The `-viewDidUnload` method was added in iOS 3.0.  Prior to that version, developers were expected to handle outlet cleanup by overriding `-setView:` and releasing outlets if the parameter was `nil`. Despite adding a clear override point for cleaning up outlets, it seems most developers were not paying attention and have neglected their view controllers by not implementing a proper `-viewDidLoad`.

[According to Apple][viewDidUnload], `-viewDidUnload` is "called when the controller’s view is released from memory."  Okay, clear as mud.  What does that really *mean*?  The ensuing paragraphs don't particularly help to understand exactly when it will be called, which seems to be a huge struggle to understand for many developers. And if developers can't envision a situation in which `-viewDidUnload` will be called they will be much more likely to skip implementing it altogether.

The most straigtforward example that demonstrates a situation that calls `-viewDidLoad` that I've found so far is using a navigation controller.  Assume that you have two view controllers A and B.  A is on screen, and in response to some event like a button press pushes B onto the navigation controller stack.  The first misconception that developers have is that this results in A being released completely and having `-dealloc` called.  That is not true.  Under ideal conditions A still exists, A's view remains loaded, and the only messages it receives are `-viewWillDisappear:` and `-viewDidDisappear:`.

Now, while view controller B is visible, a memory warning occurs.  Since A's view is not visible, it can be unloaded in response to the memory warning.  In this case, A will set its `view` property to `nil` and then call `-viewDidUnload`.  If a lazy or unwitting developer neglects to implement `-viewDidUnload` then all outlets loaded from A's nib will remain in memory despite them not being necessary and the view being `nil`.

Let's assume that example provides enough motivation that everyone now wants to implement `-viewDidUnload` and release their outlets. We now are left in the unfortunate position of having to release our outlets in both `-dealloc` *as well as* `-viewDidUnload`. This means we have more code to maintain, which means we are more likely to make mistakes in the future.  Imagine adding a new outlet to a xib; you now have to remember to add releases to two methods instead of one.  Surely there is a better way...


## Aaron Hillegass' iPhone Crap

The esteemed Aaron Hillegass of Big Nerd Ranch has [written about this problem and suggests using weak references][hillegass] for `IBOutlet` properties.  That is, declaring them as `assign` instead of `retain`:

{% highlight objc %}
    @property (nonatomic, assign) IBOutlet UIButton *crapButton;
{% endhighlight %}

Since the view controller's view will be the only owner of these outlet properties, they will automatically be released when the view controller releases its view.  Thus it nicely solves the memory management issue. However it introduces a new issues: once the view is unloaded all those weak referenced outlets are now dangling pointers to dead objects.  If you aren't careful and accidentally reference one of those properties while the view is unloaded, your app will go ***BOOM***. In a garbage collected system, those weak references will be automatically set to `nil` and this problem goes away, so when we get GC on iOS this solution will work just fine.  But in the meantime if we want to write safe code we must still set those outlets to `nil`, so this solution does not remove the burden of implementing `-viewDidUnload` in the common case.


## An Automagic Approach to Releasing Outlets

We've finally ariived at the real meat of this article. We want a solution that will not require us to manually release or nil IBOutlet properties in `-dealloc` and `-viewDidUnload`.  There are a few facts about nib loading that we can exploit to make this dream a reality.  As pointed out in [Aaron's article][hillegass], nib loading on iOS uses Key-Value Coding (KVC) to assign outlets, which means it funnels through `-setValue:forKey`.  This becomes a handy override point for tracking properties set using KVC.

{% highlight objc %}
    // excerpt, see below for full code
    - (void)setValue:(id)value forKey:(NSString *)key;
    {
        // track the property key in an NSSet
        [self.managedOutlets addObject:key];
        [super setValue:value forKey:key];
    }
{% endhighlight %}

But we don't want to track every outlet set on our object during its lifetime, only those set during nib loading. We know that `UIViewController` loads its view from a nib in the `-loadView` method, and we can override that to set up and tear down our custom property tracking so that it is only in effect during nib loading. Instead of overriding `-setValue:forKey:` directly we use a prefixed method and a small bit of runtime hackeryto swizzle it in only during the call to `-loadView`. A full explanation of method swizzling is beyond the scope of this article, but essentially we are just swapping our implementation `-zmb_setValue:forKey:` in for the real version at the start of the method, then swapping them back when the method is complete:

{% highlight objc %}
    // excerpt, see below for full code
    - (void)loadView;
    {
        // Swizzle in our version of setValue:forKey: that tracks KVO
        Method orig = class_getInstanceMethod([self class], @selector(setValue:forKey:));
        Method mine = class_getInstanceMethod([self class], @selector(zmb_setValue:forKey:));
        method_exchangeImplementations(orig, mine);
        self.managedOutlets = [NSMutableSet set];
        [super loadView];
        // put the original setValue:forKey: back like nothing ever happened.
        method_exchangeImplementations(orig, mine);
    }
{% endhighlight %}

What we end up with is a custom `UIViewController` subclass that tracks properties set using KVC during the call to `-loadView`. This custom controller can then implement a simple `-viewDidUnload` to automatically release all those properties, freeing us from doing so in a custom subclass. The complete code for the solution provided in this article is released here into the public domain.  It can also be found as part of the nascent [ZombieKit][zombiekit] project, an MIT-licensed collection of useful code from the NSZombie blog authors.

<script src="https://gist.github.com/906948.js"> </script>

At this point it is worth noting [another, very similar, solution][fpillet-magic] which comes from Florent Pillet. He also provides a `UIViewController` subclass which in `-dealloc` and `-viewDidUnload` iterates from the subclass up the inheritance chain to `UIViewController` and releases any ivars that descend from `UIView` along the way. This approach is quite clever, and actually very similar to the solution proposed in this article. His solution is slightly less generic in that it works only for properties that descend from UIView (though admittedly this will be the overwhelming majority of IBOutlets) and it does not use KVC but rather releases the ivar value directly. I won't claim either his or my solution to be superior, so feel free to make that decision for yourself.


## Caveats

[THE RULES!]: http://developer.apple.com/library/ios/#documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmRules.html
[nib mm]: http://developer.apple.com/library/ios/#documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmNibObjects.html
[viewDidUnload]: http://developer.apple.com/library/ios/documentation/uikit/reference/UIViewController_Class/Reference/Reference.html#//apple_ref/doc/uid/TP40006926-CH3-SW36
[hillegass]: http://weblog.bignerdranch.com/?p=95
[fpillet-magic]: https://gist.github.com/270266
[fpillet]: https://github.com/fpillet
[zombiekit]: https://github.com/NSZombie/ZombieKit
