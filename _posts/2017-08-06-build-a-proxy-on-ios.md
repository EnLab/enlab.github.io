In this tutorial, I'll show you how to use the versatile `URLProtocol` class to intercept network requests inside your app, independently of whether the request originated from a `URLSession`, a wrapper library such as Alamofire, an `NSURLConnection`, or an Ajax request inside a web view.

Real-world applications include: 

1. Proxies
2. Stub HTTP requests for testing (see [Mockingjay](https://github.com/kylef-archive/Mockingjay))
3. Network activity indicators (see [Big Brother](https://github.com/marcelofabri/BigBrother))
4. Certificate pinning

### Laying the Groundwork 👷

To get started, create a new **Single View Application** in Xcode, and select **Swift** as the language.

Replace the contents of `ViewController` with the following:

```swift
class ViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        
        let url = URL(string: "https://httpbin.org/post")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.addValue("application/json", forHTTPHeaderField: "Content-Type")
        request.addValue("application/json", forHTTPHeaderField: "Accept")
        request.httpBody = try? JSONSerialization.data(withJSONObject: [ "hello": "world" ], options: [])
        
        URLSession.shared.dataTask(with: request, completionHandler: { data, response, error in
            guard let data = data else { return }
            guard let str = String(data: data, encoding: .utf8) else { return }
            print(str)
        }).resume()
    }
}
``` 

The above code posts `{"hello": "world"}` (a JSON object) to a public test endpoint that simply echos the data back, along with a few other bits of information. Build and run the project, you should see something like this in your console (note that the `json` attribute contains the sent JSON):

```json
{
  "args": {}, 
  "data": "{\"hello\":\"world\"}", 
  "files": {}, 
  "form": {}, 
  "headers": {
    "Accept": "application/json", 
    "Accept-Encoding": "gzip, deflate", 
    "Accept-Language": "en-us", 
    "Connection": "close", 
    "Content-Length": "17", 
    "Content-Type": "application/json", 
    "Host": "httpbin.org", 
    "User-Agent": "TuteNSURLProtocol/1 CFNetwork/811.4.18 Darwin/16.3.0"
  }, 
  "json": {
    "hello": "world"
  }, 
  "origin": "190.104.131.250", 
  "url": "https://httpbin.org/post"
}
```

### Intercepting Requests 🕵


This is a good starting point for our proxy which will intercept outgoing network requests, wrap them in new object, and resend them on their way.

Create a new subclass of `URLProtocol` and give it a name such as `MyProxyProtocol`. Now replace its contents with the following:

```swift
class MyProxyProtocol: URLProtocol {
    
    // #1
    override class func canInit(with request: URLRequest) -> Bool {
        print(#function)
        return true
    }
    
    // #1
    override class func canonicalRequest(for request: URLRequest) -> URLRequest {
        print(#function)
        return request
    }
    
    // #3
    override func startLoading() {
        print(#function)
    }
    
    // #4
    override func stopLoading() {
        print(#function)
    }
}
```

These are the `URLProtocol` methods that a subclass must override to get the classs working:

1. `canInit(with:)` - Here we simply return `true` to indicate we want to intercept _all_ requests. A more sophisticated implementation would use the passed in `request` parameter to determine whether the request should be handled or allowed to continue (more on this later).
2. `canonicalRequest(for:)` - This method is often misunderstood [TODO]. Simply returning the request is the most common implementation.
3. `startLoading()` and `stopLoading()` - Once a request has been intercepted, `startLoading()` is called and the original request is cancelled. Here you can make a copy of the original request, modify it, and resend it. When a response is received, we communicate this to the original callee via a `URLProtocolClient` which is accessed through the `client` property.

TODO (move to App Delegate) Open `ViewController` and at the beginning of `viewDidLoad` register our new protocol handler so that it can handle HTTP requests:

```swift
class ViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        
        // Register our proxy.
        URLProtocol.registerClass(MyProxyProtocol.self)

		...
```

This is where the actual proxy is implemented, we'll first define a variable to hold our new data task.

```swift
class MyProxyProtocol: URLProtocol {
    
    var dataTask: URLSessionDataTask?
    ...
```

Now implement `startLoading()` and `stopLoading` like so:

```swift
	override func startLoading() {
	    print(#function)
	    
	    dataTask = URLSession.shared.dataTask(with: request, completionHandler: { data, response, error in
	        
	        if let data = data, let response = response {
	            self.client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
	            self.client?.urlProtocol(self, didLoad: data)
	        } else if let error = error {
	            self.client?.urlProtocol(self, didFailWithError: error)
	        }
	        self.client?.urlProtocolDidFinishLoading(self)
	    })
	    dataTask?.resume()
	    
	}
	
	override func stopLoading() {
	    self.dataTask?.cancel()
	}
```

This is basically the simplest implementation of `URLProtocol` -- that does not quite work! Give the project a run and open the console. You'll see that the `print(#function)` is being hit indefinitely in an infinite loop. Why is this so? It's because inside `startLoading()`, we're creating a new request, which again triggers `startLoading()`, which creates a new request, and on, and on...

The solution? Mark handled requests so they don't keep on getting handled. To do this, modify `canInit(with:)` and `startLoading()` like so:

```swift
    override class func canInit(with request: URLRequest) -> Bool {
        if URLProtocol.property(forKey: "is_handled", in: request) as? Bool == true {
            return false
        }
        return true
    }
    
    override func startLoading() {
        print(#function)
        
        let mutableRequest = (request as NSURLRequest).mutableCopy() as! NSMutableURLRequest
        URLProtocol.setProperty(true, forKey: "is_handled", in: mutableRequest)
        let newRequest = mutableRequest as URLRequest
        
        dataTask = URLSession.shared.dataTask(with: newRequest, completionHandler: { data, response, error in
            
            if let data = data, let response = response {
                self.client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
                self.client?.urlProtocol(self, didLoad: data)
            } else if let error = error {
                self.client?.urlProtocol(self, didFailWithError: error)
            }
            self.client?.urlProtocolDidFinishLoading(self)
        })
        dataTask?.resume()
    }
```

`URLProtocol` provides methods to add arbitrary data to `URLRequest` instances via the `setProperty(_:forKey:in:)` and get the data using `property(forKey:in:)`, and we use these methods to tag requests as handled. Now, our protocol handler is only initialized for previously unhandled requests. (On a side note, don't confuse these properties we're adding with HTTP headers, they're completely different concepts.)

There is a caveat though, see that ugly `NSMutableURLRequest` casting and copying? It turns out that the methods to set and get methods are broken as of Swift 3. `setProperty(_:forKey:in:)` inexplicatly requires an `NSMutableURLRequest` instead of a plain `URLRequest`, although if you do cast your request to `NSMutableURLRequest`, whatever you try to set is always `nil` when you retrieve it. 

The workaround is to cast the request to a Foundation equivalent (NS prefix), and make a mutable copy. Thanks to [Philippe Hausler's comments on Swift issue SR-2804](https://bugs.swift.org/browse/SR-2804?focusedCommentId=23642&page=com.atlassian.jira.plugin.system.issuetabpanels%3Acomment-tabpanel#comment-23642) for this workaround!

Finally, don't forget to pass the mutable copy instead of the original request to the data task, otherwise the property wil be set on the wrong request instance.


### Modifying Requests 🕵

To demonstrate, we'll wrap all outgoing requests in a new object. Modify the contents of `` to:

```swift
    override func startLoading() {
        print(#function)
        
        // #1
        guard let stream = request.httpBodyStream else { return }
        stream.open()
        guard let payload = try? JSONSerialization.jsonObject(with: stream, options: []) as? [String: Any] else { return }
        
        // #2
        let wrapper = [
            "data": payload,
        ]
        
        let mutableRequest = (request as NSURLRequest).mutableCopy() as! NSMutableURLRequest
        URLProtocol.setProperty(true, forKey: "is_handled", in: mutableRequest)
        var newRequest = mutableRequest as URLRequest
        
        // #3
        newRequest.httpBody = try? JSONSerialization.data(withJSONObject: wrapper, options: [])
        
        dataTask = URLSession.shared.dataTask(with: newRequest, completionHandler: { data, response, error in
            
            if let data = data, let response = response {
                self.client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
                self.client?.urlProtocol(self, didLoad: data)
            } else if let error = error {
                self.client?.urlProtocol(self, didFailWithError: error)
            }
            self.client?.urlProtocolDidFinishLoading(self)
        })
        dataTask?.resume()
    }
```

Let's run through the commented sections:

1. Specifically we want to modify the request's body, since that is where the payload of the request is at. The `httpBody` attribute of the request is `nil` here (likely because after the URL loading system converts the request to something called a stream, it empties the original body).
Accessing the body requires reading a stream, and if the body contains JSON data like in this case, it is not a difficult task with the help of `JSONSerialization`. `jsonObject(with:options:)` returns the contents of the stream as an object which we then cast to a dictionary.
2. Here we define the wrapper structure, in this case it's a trivial wrapper object.
3. The final step is to repackage the wrapped payload in the body of the new request.

Run the project and look at the console output when the reponse arrives:

```json
SENDING REQUEST: https://httpbin.org/post
startLoading()
RECEIVED RESPONSE: {
  "args": {}, 
  "data": "{\"wrapper\":{\"hello\":\"world\"}}", 
  "files": {}, 
  "form": {}, 
  "headers": {
    "Accept": "application/json", 
    "Accept-Encoding": "gzip, deflate", 
    "Accept-Language": "en-us", 
    "Connection": "close", 
    "Content-Length": "29", 
    "Content-Type": "application/json", 
    "Host": "httpbin.org", 
    "User-Agent": "TuteNSURLProtocol/1 CFNetwork/811.4.18 Darwin/16.3.0"
  }, 
  "json": {
    "wrapper": {
      "hello": "world"
    }
  }, 
  "origin": "190.104.131.250", 
  "url": "https://httpbin.org/post"
}
```

Note that the contents of the `json` attribute is now wrapped up in a `wrapper` object! In fact, any request you make will now be modified in this way. Congratulations, you've now successfully implemented a custom url handler using the `URLProtocol`.

### Extras 🕵

**Q:** My custom url protocol handler is never getting called (I'm using Alamofire). How to fix?

**A:** Alamofire doesn't use the shared (`URLSession.shared`) session under the hood, instead it uses a [seemingly equivalent setup](https://github.com/Alamofire/Alamofire/blob/8e85284b58d815b8b7ba00e887dae8a54a284018/Tests/SessionManagerTests.swift#L147), quoted here:

```swift
    let configuration = URLSessionConfiguration.default
    return URLSession(configuration: configuration, delegate: delegate, delegateQueue: nil)
```
Unfortunately, sessions created this way do not inherit protocol handlers added via `registerClass(_:)`. The workaround is to add your handlers to the configuration, via `URLSessionConfiguration`'s `protocolClasses` property. To get your protocol first in the queue to handle requests, add it to the beginning of this array. A gotcha here is that once a session is created, changing its `protocolClasses` property does nothing, you must set this property before creating the session. Putting this together, the workaround for Alamofire becomes:

```swift
class ViewController: UIViewController {

    let sharedManager: SessionManager = {
        let configuration = URLSessionConfiguration.default
        configuration.protocolClasses?.insert(MyProxyProtocol.self, at: 0)
        let manager = SessionManager(configuration: configuration)
        return manager
    }()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        sharedManager.request("https://httpbin.org/post", method: .post, parameters: [
            "foo": "bar",
        ]).responseJSON { response in
            debugPrint(response)
        }
    }
}
```

Note that `sharedManager` must be a property, otherwise if it's a local variable it will be deallocated and your request will be promptly cancelled.



**Q:** How to use multiple protocols on the same request? I want to use one protocol handler to implement certificate pinning and another to manage an activity indicator.

**A:** Since one request can only be handled by one protocol handler, we can't actually do precisely this. What we can do however, is take advantage of the fact that a _new_ request is made when `startLoading()` is called, and this new request _can_ be given to a new handler. Also, remember how we tagged requests earlier in the post to ensure they were not handled multiple times? If each of your protocols tags its handled requests, you should have no problems. Just register both protocols in your app delegate like so (the last protocol to be registered will be called first):

```swift
URLProtocol.registerClass(CertPinProtocol.self)   // second priority
URLProtocol.registerClass(ActivityProtocol.self)  // first priority
```
