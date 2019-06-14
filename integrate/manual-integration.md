# Manual Integration

{% hint style="info" %}
The Zaius Android SDK collects and associates push tokens with customers and tracks app opens corresponding to push interactions by sending the necessary events through the Zaius API from your app.
{% endhint %}

## Update build.gradle

You'll need to add the Zaius maven repository, and add the Zaius SDK as a dependency, and if you want to receive and display push notifications, you'll also need to include Google's `firebase-messaging` package.

```groovy
repositories {
  maven {
    url  "http://maven.zaius.com"
  }
}

dependencies {
  ...
  implementation 'com.zaius:androidsdk:<version>' // Add the Zaius SDK as a dependency
  implementation 'com.google.firebase:firebase-messaging:<version>' // Only required if you want to use mobile push messaging
}
```

## Initialize

Call `Zaius.start` from your `Application` implementation’s `onCreate` method:

```java
public void onCreate() {
  
  String trackerId = "abc123"; // your Zaius tracker ID  
  boolean managePushTokens = true; // send push tokens to Zaius
  boolean collectTokensWhenAnonymous = true; // collect push tokens without customer ID set
  int flushIntervalSeconds = 60; // how frequently (in seconds) to send events to Zaius
  boolean enableLogging = false; // enable SDK logs for troubleshooting
	
  zaiusConfig = new Zaius.config(managePushTokens, collectTokensWhenAnonymous, flushIntervalSeconds)
    
  Zaius.start(this, trackerId, zaiusConfig);
  
}
```

## Token Collection

To allow Zaius to start collection push tokens, you'll need to add the following lines to your `AndroidManifest.xml` file under the `<application ...>` tag:

```markup
<service
    android:name="com.zaius.androidsdk.ZaiusRegistrationIntentService"
    android:permission="android.permission.BIND_JOB_SERVICE"
    android:exported="false" />
```

Optionally the Zaius Android SDK allows you to register a receiver, to be called when Zaius retrieves a push token.

To do so you simply need to implement a class extending `ZaiusReceiver`:

```java
public class MyReceiver extends ZaiusReceiver {
  @Override
  public void onTokenRegistration(Context context, String token) {
    // Do something with the token (e.g. fire an event)
  }
}
```

Then inside your `Application` implementation in `onCreate` you should register the receiver to handle the action:

```java
    @Override
    public void onCreate() {
        // ...
        registerReceiver(
                new MyReceiver(),
                new IntentFilter(ZaiusReceiver.TOKEN_REGISTRATION_ACTION)
        );
        // ...
    }

```

## Setup Push

After you've sorted out initialization and token collection, the next step is to set up the actual Push Notifications, you'll need to extend the `FirebaseMessagingService` class, overriding the `onMessageReceived` method, and doing the majority of the logic for displaying notifications from there. 

```java
@Override
public void onMessageReceived(RemoteMessage remoteMessage) {
    sendNotification(remoteMessageToBundle(remoteMessage));
}

/**
 * Convert an FCM RemoteMessage into a Bundle
 *
 * @param remoteMessage message containing data to be translated
 * @return a Bundle containing Strings for each key in the data of the RemoteMessage
 */
private Bundle remoteMessageToBundle(RemoteMessage remoteMessage) {
    Bundle bundle = new Bundle();
    Map<String, String> data = remoteMessage.getData();
    for (String key : data.keySet()) {
        bundle.putString(key, data.get(key));
    }
    return bundle;
}

/**
 * Create and show a simple notification containing the received push data.
 *
 * @param data a Bundle of push data
 */
private void sendNotification(Bundle data) {
    Intent intent = new Intent(this, MyActivity.class); // Open this activity when the notification is clicked
    intent.putExtras(data); // Put the bundle of data from the message in here
    PendingIntent pendingIntent = PendingIntent.getActivity(this, MY_REQUEST_CODE, intent, PendingIntent.FLAG_ONE_SHOT);
    // ... etc, rest of notification code ...
}
```

For Zaius to properly record and attribute open events, you'll need to pass along the data in the `RemoteMessage` available in `FirebaseMessagingService.onMessageReceived` as extras in the `Intent` corresponding the push open itself, as shown above.

To make sure the actual open events are tracked, you'll need to call `Zaius.pushOpened` from the response activity to the above notification display, an example of an activity is shown below:

```java
public class PushActivity extends Activity {
    // ...
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // ...
        Bundle bundle = getIntent().getExtras();
        if (bundle != null) {
            try {
                Zaius.getInstance().pushOpened(bundle);
            } catch (ZaiusException e) {
                e.printStackTrace();
            }
        }
        // ...
    }
    // ...
}
```

It is absolutely essential that you call `Zaius.getInstance().pushOpened(bundle)` as shown above, and with the extra information passed from the bundle to make sure that we can track open events accurately.

{% hint style="danger" %}
#### **Zaius.pushOpened is required to track push opens**

`Zaius.pushOpened` accepts either a Firebase `RemoteMessage` or an Android `Bundle` corresponding to the notification so that metadata associated with the push can be linked to the open event for analytics and reporting in Zaius \(e.g. the Zaius campaign that sent the notification\)
{% endhint %}

{% hint style="success" %}
Pass along the `Bundle` to `Zaius.pushOpened` in the `YourActivity.onCreate` method
{% endhint %}

