---
title: API Reference

language_tabs:
  - swift
  - java
  - javascript

toc_footers:
  - <a href='https://github.com/tripit/slate'>Documentation Powered by Slate</a>

includes:

search: true
---

# Getting Started

Start a new IOS Xcode project and at the root of the project create a folder called frameworks to keep Frameworks that we will add in.

Add the following frameworks to this folder and add them in the Link Binaries With Libraries section of the project. You will need to request these from TCS/SparkCompass

* TCSAPI.framework
* Gimbal.framework (Only required for bluetooth beacons)

Import the TCSAPI SDK

```swift
//Add to the swit bridging header eg. in Project-Bridging-Header.h
#import <TCSAPI/TCSAPI.h>

```

The following background modes and data protection are required:

* Uses Bluetooth LE Accessories
* Remote Notifications
* Data Protection

# Initialization

> To initialise, use this code:

```swift
class AppDelegate: UIResponder, UIApplicationDelegate {

    func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject: AnyObject]?) -> Bool {
        TCSAppInstance.sharedInstance().appUID = "<appuid>"

        let instance:TCSAPIConnector = TCSAPIConnector.instance() as! TCSAPIConnector
        instance.appUrl = "https://<domain.sparkcompass.com>" // You will be provided a domain
        instance.baseUrl = "https://<domain.sparkcompass.com>" // You will be provided a domain

        //SDK requires local notifications to deliver certain messaging types.
        if let _ = launchOptions {
            let localnotif:UILocalNotification? = launchOptions?[UIApplicationLaunchOptionsLocalNotificationKey] as? UILocalNotification
            if (localnotif != nil) {
                TCSAppInstance.sharedInstance().receivedLocalNotification(localnotif)
            }
        }
        ...
    }
    ...
}
```

```java

```

```javascript

```

> Make sure to replace `<appuid>` with the appUid we supply you.

SparkCompass uses an appUid to allow access to the API. You will be supplied with your AppUid when we set up your account. You will also be supplied with a subdomain to connect to.

Supply the AppUid and Subdomain as shown in the code examples to initialise the SDK.

The following section describes how to connect to the platform.

<aside class="notice">
You must replace <code>`appuid`</code> with the appUid we supply you.
</aside>

# Starting the SDK

## Begin
After the app starts and the above initialization is completed, the SDK has to be started to initialize the connection with the platform.

This should occur anytime the app is started from a not running state. Ie does not have to be called everytime the App wakes from the background.

This connects to the platform and identifies this device as having access. 

```swift
//An example placed in AppDelegate
//Typically this may go elsewhere after confirmation of EULA

 func applicationDidBecomeActive(application: UIApplication) {
    //If the appInstance is not active then attempt to activate it.
    if(!TCSAppInstance.sharedInstance().isAppActive()){
        TCSAppInstance.sharedInstance().beginWithCompletion { (success: Bool, reply: String!) -> Void in
            if (success) {
                // Can now make other API calls to the platform
                // TCSAppInstance.sharedInstance().isAppActive() returns true
                // The SDK will also send a "APPINSTANCE_ACTIVE" NSNotification when ready

                
                TCSAppInstance.sharedInstance().updateRegisteredTokenApp()
            } else {
                if(reply == "EULANOTACCEPTED") {
                    self.promptToAcceptEULA() // Eula must be accepted prior to begin
                }
                if (reply == "APPLICATION_ERROR") {
                    // Some problem with the AppUID.
                }
            }
        }
    }
}
```

## AppInstance Active

>You can check whether the AppInstance has begun correctly and is ready to make API calls to the platform using the following:

```swift
TCSAppInstance.sharedInstance().isAppActive()
```

The AppInstance will post a Notification to the notification center to alert other listeners to the fact that the AppInstance is ready to go. Classes or Controllers that exist at startup and want to make requests using the AppInstance need to wait until it is ready. You can subscribe for the following notification if you want your class to be notfied when the TCSAppInstance is ready. Note this is only a consideration when the app loads new, not when it wakes from the background. <code>APPINSTANCE_ACTIVE</code>

> Register for the Nofitication <code>APPINSTANCE_ACTIVE</code> if required

```swift
NSNotificationCenter.defaultCenter().addObserver(self, 
    selector: #selector(<your-selector>), 
    name: "APPINSTANCE_ACTIVE", object: nil)
```

Also you can now make other calls to the platform, since the SDK is now initialised with a valid appInstance. If this is a new install, a new AppInstance will be created on the Platform. If this is an existing install (app restarted) or a reinstall then the existing appInstance will be used.

After connection with the platform and also everytime the wakes from the background the SDK will sync to the latest notify action definitions, menu definitions, geolocations, and beacons. It will also begin generating events automatically for various interactions - see Sending Events.

# Events and Actions
The SDK will start sending events once it is running, such as when the app is opened, when the device enters or exits a geofence, enters/exits or dwells in proximity of a beacon, scans an image target. Custom events can also be sent from your app.

Events that go to the platform can have actiontriggers attached which conditionally trigger actions based on some criteria. Actions are behaviours for the app to perform and are returned as a response to a submitted event.

To receive actions and act on them you need to set up an appActionDelegate. This should be a class or controller that persists throughout the application. Perhaps a singleton class or base ViewController.

> Set up an AppActionDelegate

```swift
TCSAppInstance.sharedInstance().appActionDelegate = <your delegate> (AppActionDelegate)
```

The AppActionDelegate should implement <code>didReceiveAppAction</code>. Actions will be returned by the platform as a response to events sent by the sdk. The actions are configured on the platform to have various types and notification behaviours.

An Action has an Action type and a Notification type. The Action Type defines the behaviour when enacted on the device, where as the Notification type defines how or if the user is Notified about its arrival.

The Action Types are as follows

*   Notification Message - A simple message Notification with Title and Message. It is presented to the user accoring to the Notification Behaviour
*   App Page - Notification Message that can redirect to another page in the app based on the Notification Type of the action. 
*   Webview with URL - Notification Message that can redirect to a webview and open a web page in the app based on the Notification Type of the action.
*   Webview with Content - Notification Message with some content that may be opened in a webview or may provide some custom input data.

The Notification Types of Actions are as follows

*   Notification with alert - actively presents an alert in the foreground or background to the user to alert them immediately and give an opportunity to confirm the action (for high priority actions). This performs the action on confirmation.
*   Slient Notification - accumulates the action silently. Can be made available to the user in a notification list when the user next uses the app (for lower priority actions). This performs the action if the user chooses to open the notification in the feed.
*   No Notification - immediately forwards the action to the app without notifying the user first. Only used in certain senarios where this is desireable. This performs the action immediately when it arrives.

A typical flow from an event to an action would be as follows. A user enters a GeoFence and the SDK sends a Geofence entry event to the platform. There are one or more ActionTriggers set up for the entry event for that geofence on the platform. Depending on the filter settings on the action triggers, one or more actions may be triggered and returned to the app. Depending on the type of action and the configuration of the action, the sdk and your app will enact the action.

> Implement <code>didReceiveAppAction</code> in your AppActionDelegate

```swift
// Example of response to actions forwarded by the SDK
-(void)didReceiveAppAction:(TCSAppAction *)action {
    [appActionLock lock]; // NSLock to keep from processing two actions simultaneously.
    switch(action.actionType) {
        case ATYPE_APPAGE:
        {
            NSString* appPage = [action getParamValueForKey:@"AppPage"];
            if([appPage isEqualToString:@"HOME"]) {
                [self goToHome:action];
            }
            if([appPage isEqualToString:@"MAP"]) {
                [self goToMap:action];
            }
            if([appPage isEqualToString:@"PROFILE"]) {
                [self goToProfile:action];
            }
            if([appPage isEqualToString:@"SETTINGS"]) {
                [self goToSettings:action];
            }
            if([appPage isEqualToString:@"CONTACT"]){
                [self sendEmail];
            }
            if([appPage isEqualToString:@"IMAGE_RECOGNITION"]){
                [self goToAR:action];
            }
            if([ appPage isEqualToString:@"SWITCH_APP"]){
                [self goToFeaturedEvents:YES];
            }
            if([appPage isEqualToString:@"EVENT"]){
                [self goToEvents];
            }
            break;
        }
        case ATYPE_WEBURL:
            NSString *url = [action getParamValueForKey:@"Url"];
            //Launch a webview with the url
            break;
        case ATYPE_WEBCONTENT:{
            NSString* htmlContent = [action getParamValueForKey:@"HtmlContent"];
            //Do Something with the content
            break;
        }
        default:
            break;
        }
    [appActionLock unlock];
}
```

#Api Extension for custom endpoints

Implement the TCSAPIExtender to add API calls that are not part of the Core SDK.

It can be convenient to make a Singlton Class where you add the implementation of the call and response implementation of the API call and then pass the data back to your app using a callback as shown.

>You can create an Api Extender as follows by using TCSApiExtender interface in both IOS and Android

```swift
class ExtenderAPI : NSObject, TCSAPIExtender {
    
    //Set up as Singleton so it can be used anywhere in your code
    static let shared: ExtenderAPI = {
       let instance = ExtendedAPI()
        return instance
    }()
    
    //Create labels for the EndPoints
    private let ExampleGetEndpoint = "ExampleGetEndpoint"
    private let ExamplePostEndpoint = "ExamplePostEndpoint"

    //Implement this function from the interface to define endpoints
    func getTCSAPICallDefinitions() -> [AnyHashable : Any] {
        var pathDefinitions = [AnyHashable: Any]()
        var definition: TCSAPICallDefinition
        
        var keyExists = pathDefinitions[ExampleGetEndpoint] != nil
        if !keyExists {
            definition = TCSAPICallDefinition(versionPath: "/api/1", path: "/example/get", andMethod: GET)
            pathDefinitions[ExampleGetEndpoint] = definition
        }

        keyExists = pathDefinitions[ExampleGetEndpoint] != nil
        if !keyExists {
            definition = TCSAPICallDefinition(versionPath: "/api/1", path: "/example/post", andMethod: POST)
            pathDefinitions[ExampleGetEndpoint] = definition
        }
        
        return pathDefinitions
    }
    

    /**
    *   Check the TCSUrlConnection class for DataCallback, ListCallback definitions.
    */
    func postExample(message: String, callback: @escaping DataCallback) {
        //Create RequestArgs object and set values
        let requestArgs = TCSRequestArgs()
        requestArgs.requestType = ExampleGetEndpoint
        
        var itemDictionary = [AnyHashable: Any]()
        
        //Set any POST data structure to a dictionary
        itemDictionary["message"] = message
        
        requestArgs.payload = itemDictionary
        //Supply a callback
        requestArgs.callback = {
            (connection, httpCode, data, error) in

            //Process the response
            if error == nil && httpCode == 200 {
                do {
                    let itemDictionary = try JSONSerialization.jsonObject(with: data!, options: .mutableContainers) as! [AnyHashable : Any]
                    
                    let template = itemDictionary
                    callback(template, nil)
                } catch let anError as NSError {
                    callback(nil, anError)
                }
            } else {
                callback(nil, error)
            }
        }
        
        (TCSAPIConnector.instance() as! TCSAPIConnector).request(with: requestArgs)
    }
    
    func getExample(callback: @escaping ListCallback) {
        let requestArgs = TCSRequestArgs()
        requestArgs.requestType = ExampleGetEndpoint
        requestArgs.payload = [AnyHashable : Any]()
        
        requestArgs.callback = { (connection, httpCode, data, error) in
            var error = error
            var rewards = [AnyObject]()
            if error == nil && httpCode == 200 {
                do {
                    let itemDictionary = try JSONSerialization.jsonObject(with: data!, options: .mutableContainers) as! [AnyHashable : Any]
                    rewards = itemDictionary["rewards"] as! [AnyObject] 
                } catch let anError as NSError {
                    error = anError
                }
            }
            callback(rewards, error)
        }
        
        (TCSAPIConnector.instance() as! TCSAPIConnector).request(with: requestArgs)
        
    }
}
```

```java
//Create a class for extending the API
public class ExtenderAPI implements TCSAPIExtender {

    private static ExtenderAPI instance;
    public static final String ERROR_REQUEST_FAILED = "Error_Request_Failed";

    public static final String GETEXAMPLE = "GetExample";
    public static final String POSTEXAMPLE = "PostExample";

    public static ExtenderAPI getInstance() {
        if (instance == null) {
            instance = new ExtenderAPI();
        }
        return instance;
    }

    @Override
    public Map<String, TCSAPICallDefinition> getTCSAPICallDefinitions() {

        Map<String, TCSAPICallDefinition> pathDefinitions = new HashMap<>();
        TCSAPICallDefinition definition;

        if (!pathDefinitions.containsKey(GETRREVENTS)) {
            definition = new TCSAPICallDefinition("/api/1", "/events", TCSAPIConstants.TCSRequestMethod.GET);
            pathDefinitions.put(GETRREVENTS, definition);
        }

        if (!pathDefinitions.containsKey(POSTMISSEDPOINTSREQUEST)) {
            definition = new TCSAPICallDefinition("/api/1", "/points/missedpointsrequest", TCSAPIConstants.TCSRequestMethod.POST);
            pathDefinitions.put(POSTMISSEDPOINTSREQUEST, definition);
        }

        return pathDefinitions;
    }

    public void postExample(String message, final TCSDataReturnedDelegate delegate) {
        TCSRequestArgs requestArgs = new TCSRequestArgs();
        requestArgs.requestType = ExtenderAPI.POSTEXAMPLE;

        JSONObject values = new JSONObject();
        try {
            values.put("message", message);
        } catch (JSONException e) {
            e.printStackTrace();
        }

        requestArgs.payload = values;
        requestArgs.callback = new TCSURLConnectionDelegate() {
            @Override
            public void done(TCSURLConnection connection, int responseCode, String reply, Exception e) {
                JSONObject data = null;
                Error error = null;

                if (e != null) {
                    error = new Error(e.getMessage(), e);
                } else {
                    try {
                        if (responseCode == 200) {
                            data = new JSONObject(reply);
                        } else {
                            error = new Error(ERROR_REQUEST_FAILED, null);
                        }
                    } catch (JSONException e1) {
                        error = new Error(ERROR_REQUEST_FAILED, e1);
                    }
                }
                if (delegate != null) {
                    delegate.done(data, error);
                }
            }
        };

        //Submit the request using TCSApiConnector
        try {
            TCSAPIConnector.getInstance().requestWithArgs(requestArgs);
        } catch (Exception e) {
            delegate.done(null, new Error(e.getMessage(), e));
        }
    }

    public void getExample(final TCSDataReturnedDelegate delegate) {
        TCSRequestArgs requestArgs = new TCSRequestArgs();
        requestArgs.requestType = ExtenderAPI.GETEXAMPLE;
        requestArgs.callback = new TCSURLConnectionDelegate() {
            @Override
            public void done(TCSURLConnection connection, int responseCode, String reply, Exception e) {
                JSONObject data = null;
                Error error = null;

                if (e != null) {
                    error = new Error(e.getMessage(), e);
                } else {
                    try {
                        data = new JSONObject(reply);
                        if (responseCode == 200) {

                        } else {
                            error = new Error(ERROR_REQUEST_FAILED, null);
                        }
                    } catch (JSONException e1) {
                        error = new Error(ERROR_REQUEST_FAILED, e1);
                    }
                }
                if (delegate != null) {
                    delegate.done(data, error);
                }
            }
        };

        try {
            TCSAPIConnector.getInstance().requestWithArgs(requestArgs);
        } catch (Exception e) {
            delegate.done(null, new Error(e.getMessage(), e));
        }
    }
```

>Add the ApiExtender to TCS

```swift
// In AppDelegate:
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
    //... 
    (TCSAPIConnector.instance() as! TCSAPIConnector).extendsAPI(ExtenderAPI.shared)
}
```
```java
//Add when initialising TCSAPI AppUrl and ApiKey
TCSAPIConnector.getInstance().extendAPI(ExtenderAPI.getInstance());
```

>Use the APIExtensions in your code

```swift
ExtenderAPI.shared.getExample(callback: { (list, error) in
    if error != nil || list == nil {
        return
    }
    
    for item in list! {

    }
})
```
```java
ExtenderAPI.getInstance().getExample( new TCSDataReturnedDelegate() {
    @Override
    public void done(JSONObject data, Error error) {
        if (error == null) {
            // Do something with data object
        } else {
            //Error Handling
        }
    }
});
```




#Platform Requests

Requests can be made to the platform to sync various items like calendars, events, exhibitors/speakers, Banner menus for the home screen.











