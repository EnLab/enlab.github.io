---
layout: post
title: Popups in Swift 
---

Here's a how-to on making custom popups in Swift for iOS. In the process, you'll learn how to use the `UIWindow` class and static variables.

### What We Want

{% highlight swift %}
// Usage
Popup(title: "Deleting all cat photos from your phone...") {
    print("Ok tapped")
}.show()
{% endhighlight %}

<img src="{{ site.url }}/assets/screenshot.png" alt="Screenshot" style="width: 200px;"/>

Simple straight forward syntax for calling the popup, _without creating a local variable_. This will help reduce code clutter and make for neater syntax. 

### Get Started

Create a class, let's say `Popup`, that will encapsulate the popup's functionality. The `window` will allow the popup to cover the entire screen, including navigation bars, etc.

{% highlight swift %}
class Popup: NSObject {
    var window: UIWindow!

}
{% endhighlight %}

Why inherit from `NSObject`? It will be necessary later for the class to be compatible with Obj-C when we add `UIGestureRecognizer` protocol conformance.

Add the following code inside `Popup` to define its public interface and create the `window`.

{% highlight swift %}
init(title: String, positiveCallback: (Void -> Void)? = nil) {
    super.init()

    // A UIWindow does not need to be added via addSubview
    // it appears on the screen when its hidden property is set to false.
    window = UIWindow()
    window.backgroundColor = UIColor.blackColor().colorWithAlphaComponent(0.3)
    window.windowLevel = UIWindowLevelAlert 
}

func show() {
    // Animate window from transparent to opaque.
    window.alpha = 0
    window.hidden = false   // Adds window to the screen.

    UIView.animateWithDuration(0.3, animations: {
        self.window.alpha = 1
    }) 
}
{% endhighlight %}

The `show` function sets the `window`'s `hidden` property to `true` to make it visible, and the animation block animates the window from invisible to visible.

Add a button to the screen to test the popup. The button's action should simply call the Popup's initializer method:

{% highlight swift %}
@IBAction func action(sender: AnyObject) {
    Popup(title: "Deleting all cat photos from your phone...") {
        print("Ok tapped!")
    }.show()
}
{% endhighlight %}

Give it a run. What happens?

Absolutely nothing.

What's wrong? The problem is that we are not keeping any reference to the popup, so it is being deallocated when the `init` method finishes, before the `show`  method is called. One possible fix is to keep a reference to the `Popup` in the calling class:

{% highlight swift %}
var popup: Popup!

@IBAction func action(sender: AnyObject) {
    popup = Popup(title: "Deleting all cat photos from your phone...") {
        print("Ok tapped!")
    }
    popup.show()
} 
{% endhighlight %}

Try this now. It works, but it complicates our code, creating an instance variable and calling `show` on another line. The solution is to create a static variable on the `Popup` class, which retains a reference to the current popup. Since there will only ever be one popup shown at a time, there will never be any problem due to multiple instances of the popup in memory.

Remove the modified syntax of the `Popup` initializer, and add the following code inside `Popup`'s initializer, after `super.init()`:

{% highlight swift %}
init(title: String, positiveCallback: (Void -> Void)? = nil) {
    super.init()
    Popup.popup = self
    ...
{% endhighlight %}

Run the project again, and a semi-transparent popup should cover the entire screen when the button action is called.

## Adding The Popup Contents

The contents for this example are a title and OK button, add these to the end of the `init` initializer:

{% highlight swift %}
// The background view is centered and its size height is dynamic, depending on its contents.
let backgroundView = UIView()
backgroundView.backgroundColor = UIColor.whiteColor()
backgroundView.translatesAutoresizingMaskIntoConstraints = false
window.addSubview(backgroundView)
window.addConstraints(NSLayoutConstraint.constraintsWithVisualFormat("H:|-(40)-[backgroundView]-(40)-|", options: NSLayoutFormatOptions(rawValue: 0), metrics: nil, views: ["backgroundView": backgroundView]))
window.addConstraint(NSLayoutConstraint(item: backgroundView, attribute: .CenterY, relatedBy: .Equal, toItem: window, attribute: .CenterY, multiplier: 1, constant: 0))
backgroundView.layer.cornerRadius = 8
backgroundView.layer.masksToBounds = true

// The title label, not much happening here.
let titleLabel = UILabel()
titleLabel.text = title
titleLabel.translatesAutoresizingMaskIntoConstraints = false
backgroundView.addSubview(titleLabel)
titleLabel.numberOfLines = 0
titleLabel.lineBreakMode = .ByWordWrapping
titleLabel.textAlignment = .Center
backgroundView.addConstraints(NSLayoutConstraint.constraintsWithVisualFormat("H:|-[titleLabel]-|", options: NSLayoutFormatOptions(rawValue: 0), metrics: nil, views: ["titleLabel": titleLabel]))


// The OK button.
let button = UIButton()
backgroundView.addSubview(button)
button.translatesAutoresizingMaskIntoConstraints = false
backgroundView.addConstraints(NSLayoutConstraint.constraintsWithVisualFormat("H:|-[button]-|", options: NSLayoutFormatOptions(rawValue: 0), metrics: nil, views: ["button": button]))
button.setTitle("Ok", forState: .Normal)
button.setTitleColor(UIColor.whiteColor(), forState: .Normal)
button.backgroundColor = UIColor.greenColor()
button.layer.cornerRadius = 8
button.layer.masksToBounds = true
button.addTarget(self, action: #selector(positiveButtonCallback), forControlEvents: .TouchUpInside)

// The vertical placement of the title and button. This gives the background view its height.
let views = ["titleLabel": titleLabel, "button": button]
backgroundView.addConstraints(NSLayoutConstraint.constraintsWithVisualFormat("V:|-[titleLabel]-[button]-|", options: NSLayoutFormatOptions(rawValue: 0), metrics: nil, views: views))
{% endhighlight %}

You'll get some errors which we'll fix rightaway. Add these methods to the `Popup`:

{% highlight swift %}
func close() {
    // Animate window from opaque to transparent.
    UIView.animateWithDuration(0.3, animations: {
        self.window.alpha = 0
    }, completion: { _ in
        Popup.popup = nil
    })
}

func dismissAction() {
    close()
}

func positiveButtonCallback() {
    positiveCallback?()
    close()
}
{% endhighlight %}

Next, add this variable to the top of the `Popup` class:

{% highlight swift %}
var positiveCallback: (Void -> Void)?
{% endhighlight %}

Run the code, and you should have a working popup. Questions or comments? Please leave your them below.
