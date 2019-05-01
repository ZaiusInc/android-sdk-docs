# Events

Track customer activity via events.

```java
ZaiusEvent genericEvent = new ZaiusEvent("my_event_type")
  .action("my_action")
  .addField("some_other_key", "some_custom_field_value");
Zaius.sendEvent(genericEvent);
```



