# Manual Integration

{% hint style="info" %}
The Zaius Android SDK collects and associates push tokens with customers and tracks app opens corresponding to push interactions by sending the necessary events through the Zaius API from your app.
{% endhint %}

## Update build.gradle

```groovy
repositories {
  maven {
    url  "http://maven.zaius.com"
  }
}

dependencies {
  ...
  compile 'com.zaius:androidsdk:<version>' // Add the Zaius SDK as a dependency
  compile 'com.google.firebase:firebase-messaging:<version>' // Only required if you want to use mobile push messaging
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

## Setup Push

Pass along the data in the `RemoteMessage` available in `FirebaseMessagingService.onMessageReceived` as extras in the `Intent` corresponding to push opens:

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

{% hint style="danger" %}
#### **Zaius.pushOpened is required to track push opens**

`Zaius.pushOpened` accepts either a Firebase `RemoteMessage` or an Android `Bundle` corresponding to the notification so that metadata associated with the push can be linked to the open event for analytics and reporting in Zaius \(e.g. the Zaius campaign that sent the notification\)
{% endhint %}

{% hint style="success" %}
Pass along the `Bundle` to `Zaius.pushOpened` in the `YourActivity.onCreate` method
{% endhint %}

## Register for Registration Callbacks

#### Optional Step

The Zaius Mobile SDK for Android supports event callbacks. Push token registration is supported for clients who want to be notified of push token registration events.

Implement a class extending `ZaiusReceiver`:

```java
public class MyReceiver extends ZaiusReceiver {
  @Override
  public void onTokenRegistration(Context context, String token) {
    // Do something with the token (e.g. fire an event)
  }
}
```

Reference the receiver inside the `application` element of your AndroidManifest.xml:

```markup
<receiver android:name=".MyReceiver" exported="false">
  <intent-filter>
    <action android:name="com.zaius.androidsdk.TOKEN_REGISTRATION_ACTION" />
  </intent-filter>
</receiver>
```

