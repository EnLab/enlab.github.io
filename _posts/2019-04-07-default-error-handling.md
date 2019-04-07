---
layout: post
title: Default Network Error Handling in Swift 
---


This is a pattern I usually use when making network requests in Swift that keeps error handling in one place while allowing me to provide custom error handling where I need it.

I find it very useful to _not_ provide an explicit error handler in every network request. Since most requests should handle errors the same way, moving this code to a default handler seems a logical step. This reduces duplicate code and increases maintainability, since there will be only one spot in the code base to modify when changes need to be made.

The following example shows how I create a wrapper around Alamofire to provide a place to put default error handling. The example is simplified to only handle a specific type of GET request.

```swift
enum HTTP {
    
    typealias OnSuccess = ((Data) -> Void)
    typealias OnError = ((Data?) -> Bool)
    
    static func get(url: String, onSuccess: OnSuccess? = nil, onError: OnError? = nil) {
        
        let dataRequest = Alamofire.request(url)
        
        // Handle generic errors here.
        // This could log the error, for example.
        let defaultErrorHandler = {
            print("Default error handler called...")
        }
        
        func handleError(_ data: Data?) {

            // If an custom error handler was provided, evaluate it.
            if let customErrorHandler = onError {
                let wasHandled = customErrorHandler(data)

                // If the custom error handler failed to handle the error, call the default implementation
                if wasHandled == false {
                    defaultErrorHandler()
                }
            } else {
                // Since no custom error handler was provided, call the default.
                defaultErrorHandler()
            }
        }

        dataRequest.responseData { (response: DataResponse<Data>) in
            
            // Handle errors
            guard response.error == nil else {
                handleError(response.data)
                return
            }
            
            // Handle non-HTTP responses
            guard let httpURLResponse = response.response, httpURLResponse.statusCode == 200 else {
                handleError(response.data)
                return
            }
            
            // Handle nil data
            guard let data = response.data else {
                handleError(nil)
                return
            }
            
            onSuccess?(data)
        }
    }
}

```

This wrapper can be called in a variety of ways depending on the use case:

#### Default error handler
Fire the request and leave any errors to the default error handler.
```swift
HTTP.get(url: "https://httpbin.org/get", onSuccess: { data in
    print("On success!")
})
```
#### Custom error handler
This is the situation where we know of some specific errors that can occur and we want to catch them if they show up. In the example below, `canHandleError` would be logic that checks the error and returns `true` if the error is known, `false` otherwise.
Returning `true` tells the wrapper that the error was handled successfully, while `false` means the error was not handled.

```swift
HTTP.get(url: "https://httpbin.org/get", onSuccess: { data in
    print(data.count)
}, onError: { data in
    // Do error handling here
    if canHandleError {
        print("Error handled")
        return true
    } else {
        print("The error was not one we're expecting")
        return false
    }
})
```
