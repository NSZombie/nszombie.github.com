---
layout: post
title: Constraint Based Layout for iOS
---

# {{ page.title }}

CoreAnimation on the desktop has had the concept of layout managers since its introduction in 10.5, but iOS missed out on this great feature; instead having to resort to absolute positioning. 

I wanted to try to bring a similar system to iOS, but instead of restricting it to CALayers, allowing it to be used with any UIView.

The results of that experiment can be seen [here](https://github.com/jweinberg/JWLayoutViews/)

In this post I will try to explain how to use it.

## Using JWConstraintLayoutView

Here is a simple example of laying out two views, and applying some constraints to them.
{% highlight objectivec %}
//This is just here to provide us a reference so the rest of this example makes sense
JWConstraintLayoutView *layoutView = nil;
layoutView = [[JWConstraintLayoutView alloc] initWithFrame:CGRectMake(0,0,320,480)];

UIView *viewA = [[UIView alloc] initWithFrame:CGRectZero];

viewA.frame = CGRectMake(0.0,0.0,100.0,25.0);
viewA.backgroundColor = [UIColor redColor];
[layoutView addConstraint:[JWConstraint constraintWithView:viewA
                                                 attribute:kJWConstraintMidY
                                                relativeTo:nil
                                                 attribute:kJWConstraintMidY]];

[layoutView addConstraint:[JWConstraint constraintWithView:viewA
                                                 attribute:kJWConstraintMidX
                                                relativeTo:nil
                                                 attribute:kJWConstraintMidX]];

[layoutView addSubview:viewA];

UIView *viewB = [[UIView alloc] init];
viewB.backgroundColor = [UIColor greenColor];
[layoutView addConstraint:[JWConstraint constraintWithView:viewB
                                                 attribute:kJWConstraintWidth
                                                relativeTo:viewA
                                                 attribute:kJWConstraintWidth]];

[layoutView addConstraint:[JWConstraint constraintWithView:viewB
                                                 attribute:kJWConstraintMidX
                                                relativeTo:viewA
                                                 attribute:kJWConstraintMidX]];


[layoutView addConstraint:[JWConstraint constraintWithView:viewB
                                                 attribute:kJWConstraintMinY
                                                relativeTo:viewA
                                                 attribute:kJWConstraintMaxY
                                                    offset:10.0]];

[layoutView addConstraint:[JWConstraint constraintWithView:viewB
                                                 attribute:kJWConstraintMaxY
                                                relativeTo:nil
                                                 attribute:kJWConstraintMaxY
                                                    offset:-10.0]];

[layoutView addSubview:viewB];
{% endhighlight %}

The end result will look something like this   
![Constraints](/images/constraints-1.png "Constraints")

As you can see the top view (viewA) is centered in the screen, and viewB is positioned below it exactly as described in the constraints above. This gives us some very powerful options, as these will get reapplied when either the parent or any sibling view changes size. Meaning that when the device is rotated the constraints are still valid and allows for the creation of some very fluid layouts.

A full and much more complicated example can be found [here.](https://github.com/jweinberg/JWLayoutViews/blob/master/Classes/ConstraintLayoutTestViewController.m)

This example is a direct translation of Apple's [CAConstraintLayoutManger example](http://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CoreAnimation_guide/Articles/Layout.html#//apple_ref/doc/uid/TP40006084-SW5). Other than the obvious API differences, the only real change is modifying which constraints are set on viewB, as CoreAnimation and UIKit have flipped coordinate spaces.

The provided API is boils down to JWConstraint, these objects are constructed and added to the layout view as illustrated in the previous sample.

{% highlight objc %}
- (id)initWithView:(UIView*)aView
         attribute:(JWConstraintAttribute)anAttribute 
        relativeTo:(UIView*)aRelativeView 
         attribute:(JWConstraintAttribute)aRelativeAttribute 
             scale:(CGFloat)aScale 
            offset:(CGFloat)aOffset;
{% endhighlight %}


