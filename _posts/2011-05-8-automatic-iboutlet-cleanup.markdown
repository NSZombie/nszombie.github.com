---
author: Jason Foreman
title: Automatic IBOutlet Cleanup
layout: post
tags: ['iOS', 'UIKit']
---

Memory management is a common and constant source of headaches and
confusion among iOS developers. Even after mastering the basic [memory
management rules][THE RULES!], many developers fail to properly manage
IBOutlets across the entire view controller lifetime, particularly with
regards to view unloading. In this article we examine that problem in
detail and present a solution to make IBOutlet management much easier
and less prone to error. With a small UIViewController subclass and
just the tiniest bit of Objective-C runtime manipulation we'll get
automatic IBOutlet cleanup in `viewDidLoad` as well as `dealloc`.


## View Controller and IBOutlet Lifetime

[Apple's documentation recommends][nib mm] using a nonatomic, retained
property for implementing IBOutlets on iOS. This means that every object
that is loaded from a nib and assigned to such a property will remain in
memory until is is released by the view controller, typically by setting
the property to `nil`. Everyone knows--or should know--that it is
important to release outlets in the `-dealloc` method (though exactly
how they are released is the topic of [some][lamarche nil]
[debate][jalkut nil]).
Unfortunately many developers, even experienced ones, overlook or
misunderstood the importance of the `-viewDidUnload` method.

### Frackin -viewDidUnload, how does it work?

The `-viewDidUnload` method was added in iOS 3.0.  Before it was added
developers were expected to handle outlet cleanup in response to memory
warnings by overriding `-setView:` and releasing outlets if the
parameter was `nil`. Despite adding a clear override point for cleaning
up outlets, it seems most developers were not paying attention and have
neglected their view controllers by not implementing a proper
`-viewDidLoad`.

[According to Apple][viewDidUnload], `-viewDidUnload` is "called when
the controller’s view is released from memory." OK, but what does that
really *mean*? It seems that a great many developers are unable to
figure out from the documentation just when `-viewDidUnload` will be
called, and if they can't envision a situation in which `-viewDidUnload`
will be called they will be much more likely to skip implementing it
altogether.

The most straigtforward example that demonstrates `-viewDidLoad` is when
using a navigation controller.  Assume that you have two view
controllers A and B.  A is on screen, and in response to some event like
a button press controller B is pushed onto the navigation stack.  The
first misconception that developers have is that this results in A being
released completely and having `-dealloc` called.  That is not true.
Under ideal conditions A still exists, A's view remains loaded, and the
only messages it receives are `-viewWillDisappear:` and
`-viewDidDisappear:`.

Now assume that while view controller B is visible a memory warning
occurs.  Since A's view is not visible it can be unloaded in response
to the memory warning.  In this case, A will set its `view` property to
`nil` and then call `-viewDidUnload`.  If a lazy or unwitting developer
neglects to implement `-viewDidUnload` then all outlets loaded from A's
nib will remain in memory despite them not being necessary and the view
being `nil`.

We now understand when `-viewDidUnload` is called and why it is
important to handle it. Unfortunately this means we now have *2* places
to release every outlet: `-viewDidUnload` and of course `-dealloc`. This
quickly becomes a maintenance nightmare if a class has more than
a couple of outlets, so we immediately want an easier way.  That's
exactly what we'll build later, but first we'll look at an alternative
suggestion that partially solves this problem.


## Aaron Hillegass' iPhone Crap

The esteemed Aaron Hillegass of Big Nerd Ranch has [written about this
problem][hillegass] and suggests using weak references for `IBOutlet`
properties.  That is, declaring them as `assign` instead of `retain`:

{% highlight objc %}
@property (nonatomic, assign) IBOutlet UIButton *crapButton;
{% endhighlight %}

Since the view controller's view will be the only owner of these outlet
properties, they will automatically be released when the view controller
releases its view.  Thus it nicely solves the memory management issue.
However it introduces a new issue: once the view is unloaded all those
weak referenced outlets are now dangling pointers to dead objects.  If
you aren't careful and accidentally reference one of those properties
while the view is unloaded, your app will go ***BOOM***.

Now, in a garbage collected system those weak references will be
automatically set to `nil` and this problem goes away. When we get
garbage collection in iOS this solution will work just fine, and in fact
probably become the preferred approach.  But in the meantime if we want
to write safe code we must still set those outlets to `nil`, so this
solution does not remove the burden of implementing `-viewDidUnload` in
the common case. In the next section we'll finally see a solution that
solves the problem completely under manual memory management.


## An Automagic Approach to Releasing Outlets

What we want is a solution that will not require us to manually release
or set to nil IBOutlet properties in `-dealloc` and `-viewDidUnload`.
There are a few facts about nib loading that we can exploit to make this
dream a reality.  As pointed out in [Aaron's article][hillegass], nib
loading on iOS uses Key-Value Coding (KVC) to assign values, which means
it will call `-setValue:forKey:`.  We'll exploit this fact in order to
track outlets that are set during nib loading so that they can later be
automatically released when necessary.

{% highlight objc %}
// excerpt, see below for full code
- (void)setValue:(id)value forKey:(NSString *)key;
{
    // track the property key in an NSSet
    [self.managedOutlets addObject:key];
    [super setValue:value forKey:key];
}
{% endhighlight %}

Now we don't want to track every outlet set on our object during its
lifetime, only those set during nib loading. We know that
`UIViewController` loads its view from a nib in the `-loadView` method,
so we can simply override that to set up and tear down our custom
property tracking, that way it is only in effect during nib loading.
Instead of overriding `-setValue:forKey:` directly we use a prefixed
method and a small bit of runtime hackery to swizzle it in only during
the call to `-loadView`. A full explanation of method swizzling is
beyond the scope of this article, but essentially we are just swapping
our implementation `-zmb_setValue:forKey:` in for the real version at
the start of the method, then swapping them back when the method is
complete:

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

What we end up with is a custom `UIViewController` subclass that tracks
properties set using KVC during the call to `-loadView`. This custom
controller can then implement a simple `-viewDidUnload` to automatically
release all those properties, freeing us from doing so in a custom
subclass. When creating a new view controller simply subclass
`ZMBViewController` instead of `UIViewController` to opt-in to this
automatic functionality.

The complete code for the solution provided in this article is
released here into the public domain.  It can also be found as part of
the nascent [ZombieKit][zombiekit] project, an MIT-licensed collection
of useful code from the NSZombie blog authors.

<script src="https://gist.github.com/961836.js"> </script>

[THE RULES!]: http://developer.apple.com/library/ios/#documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmRules.html
[nib mm]: http://developer.apple.com/library/ios/#documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmNibObjects.html
[viewDidUnload]: http://developer.apple.com/library/ios/documentation/uikit/reference/UIViewController_Class/Reference/Reference.html#//apple_ref/doc/uid/TP40006926-CH3-SW36
[hillegass]: http://weblog.bignerdranch.com/?p=95
[zombiekit]: https://github.com/NSZombie/ZombieKit
[lamarche nil]: http://iphonedevelopment.blogspot.com/2010/10/outlets-cocoa-vs-cocoa-touch.html
[jalkut nil]: http://www.red-sweater.com/blog/1423/dont-coddle-your-code
