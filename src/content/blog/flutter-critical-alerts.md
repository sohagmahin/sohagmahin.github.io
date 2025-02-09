---
title: Implementing Critical Alerts with Firebase Notifications in Flutter for Android and iOS
author: Sohag
pubDatetime: 2025-02-09T09:49:58Z
slug: implementing-critical-alerts-with-firebase-notifications-in-flutter-for-android-and-ios
featured: true
draft: false
tags:
  - Flutter
  - Android
  - IOS
  - Critical-alerts
description: "Learn how to implement critical alerts using Firebase Notifications in Flutter for both Android and iOS. This guide covers setup, permissions, and best practices to ensure users receive high-priority notifications even in Do Not Disturb mode."
---

> This article is originally from my [blog post](https://medium.com/@sohagmahin/implementing-critical-alerts-with-firebase-notifications-in-flutter-for-android-and-ios-93064afc2f07).

Critical Alerts: Never Miss Important Notifications

![Critical Alerts banner](https://github.com/user-attachments/assets/d3e788f9-79eb-4472-9d20-46880bb99688)

## Table of contents

## Intro

Critical Alerts are a unique type of notification designed to bypass Do Not Disturb, silent, and focus modes. This ensures you receive crucial updates and alerts, no matter the current mode of your phone.

## IOS

Implementing critical alerts on iOS is simple because it’s a built-in feature, introduced in iOS 12 (September 2018). However, It needs to get approval from Apple before using it.

To start, must request permission for `Critical Alerts` through Apple’s developer portal. You can submit your request here: [Critical Alerts Entitlement Request.](https://developer.apple.com/contact/request/notifications-critical-alerts-entitlement/)

After submitting the request, Apple will approve it in a few days. Once approved, we can add critical alerts to the iOS app.

![Critical message letter](https://github.com/user-attachments/assets/34234ccb-28d5-418e-8e94-ae771f13327c)

### Setup Notification Entitlements

Here is the details guide on how to set up critical alerts entitlements.

[iOS: How Do You Implement Critical Alerts for Your App When You Don't Have an Entitlements File?](https://stackoverflow.com/questions/66057840/ios-how-do-you-implement-critical-alerts-for-your-app-when-you-dont-have-an-en?source=post_page-----93064afc2f07--------------------------------)

### Requesting permission for critical notification

In your Flutter app, request permission for critical notifications by using the `flutter_critical_alert_permission_ios` package:

```dart
    if (Platform.isIOS) {
      FlutterCriticalAlertPermissionIos.requestCriticalAlertPermission();
    }
```

Once you completes these steps, The app will show a critical alert prompt, allowing users to enable critical alerts on their devices.

![Critical alerts permission](https://github.com/user-attachments/assets/2c1e8641-b555-4592-94f2-4de7e50a5252)

**_Voilà_**, you are now ready to receive critical alert notifications on the iOS system.

## Android

Unlike iOS, Android does not natively support critical alerts. However, you can implement a workaround by utilizing the `Do Not Disturb (DND)` permissions and manipulating the device’s notification settings.

### 1. Request Do Not Disturb Permission

To modify Do Not Disturb (DND) settings on Android, you need to request permission using the `DoNotDisturbPlugin`

```dart

  static Future<void> requestDoNotDisturbPermission(
      BuildContext context) async {
    final dndPlugin = DoNotDisturbPlugin();
    bool hasAccess = await dndPlugin.isNotificationPolicyAccessGranted();

    if (!hasAccess) {
      showDialog(
        context: context,
        builder: (BuildContext context) {
          return CupertinoAlertDialog(
            title: const Text('Allow this app to access Do Not Disturb'),
            content: const Text(
                'This will allow the app to manage Do Not Disturb settings'),
            actions: [
              TextButton(
                onPressed: () {
                  Navigator.of(context).pop();
                },
                child: const Text("Cancel"),
              ),
              TextButton(
                onPressed: () async {
                  await dndPlugin.openNotificationPolicyAccessSettings();
                  Navigator.of(context).pop();
                },
                child: const Text("Open Settings"),
              ),
            ],
          );
        },
      );
    }
  }

```

This code essentially checks whether the permission has already been granted. If the permission has not been granted, it displays a dialog prompting the user to open the settings and enable the `Do Not Disturb` access permission, as shown below.

![DND Turn On|Off switch](https://github.com/user-attachments/assets/f74a8f41-3093-4d45-a3f2-6ea8eecc3ab0)

### 2. Manage Critical Notifications

In your notification handling, check if the notification is critical by checking a flag, and then manage DND permissions accordingly. Here’s how you can handle it:

```dart
if (message.data["critical"] == "true") {
      if (Platform.isAndroid) {
        CriticalNotificationManager().manageCriticalNotificationAccess();
      }
    }
```

And then `manageCriticalNotificationAccess` method ensures that the critical notification bypasses any DND or silent settings.

### 3. Overwrite DND mode.

If the app has access to modify DND settings, you can enable or disable interruptions:

```dart
    bool hasAccess = await dndPlugin.isNotificationPolicyAccessGranted();

    if (!hasAccess) {
    } else {
      final isDndEnabled = await dndPlugin.isDndEnabled();

      if (isDndEnabled) {
        await dndPlugin.setInterruptionFilter(InterruptionFilter.all);

        Future.delayed(const Duration(seconds: 5), () async {
          await dndPlugin.setInterruptionFilter(InterruptionFilter.none);
        });

        return;
      } else {
        if (Platform.isAndroid) {
          Sound().temporarilySwitchToNormalMode();
        }
      }
    }
```

This code checks if the device is in DND mode. If it is, it temporarily disables DND by setting the interruption filter to “all,” allowing critical notifications. After a few seconds, it restores the previous mode. If DND is not enabled, it checks for silent mode and overrides it if necessary.

```dart
  void temporarilySwitchToNormalMode() async {
    RingerModeStatus ringerStatus = await SoundMode.ringerModeStatus;
    if (ringerStatus == RingerModeStatus.silent) {
      await SoundMode.setSoundMode(RingerModeStatus.normal);
      Future.delayed(const Duration(seconds: 5), () async {
        await SoundMode.setSoundMode(ringerStatus);
      });
    }
  }
```

This function checks the current ringer mode of a device. If the device is in `silent mode`, it temporarily switches the mode to `normal` for 5 seconds and then reverts it back to the original `silent mode`.

Now your app is ready for getting critical notifications.

Use this Python script to trigger critical messages through FCM.

```yaml
import firebase_admin
from firebase_admin import credentials, messaging
from decouple import config

service_account_path = "/Users/sohag/Desktop/critical-alert-firebase-adminsdk.json"

cred = credentials.Certificate(service_account_path)
firebase_admin.initialize_app(cred)

def send_notification_to_all(message_title, message_body):
    # Create a message to send to all users
    message = messaging.Message(
        notification=messaging.Notification(),
        topic='all',
        data={
            "title": message_title,
            "body": message_body,
            "critical": "true"
        },
        apns=messaging.APNSConfig(
            payload=messaging.APNSPayload(
                aps=messaging.Aps(
                    alert=messaging.ApsAlert(
                        title=message_title,
                        body=message_body
                    ),
                    badge=1,
                    sound=messaging.CriticalSound(
                        name="critical_alert.wav",
                        critical=1,
                        volume=1.0
                    )
                )
            )
        ),
    )

    # Send the notification
    try:
        response = messaging.send(message)
        print(f'[NOTIFICATION-SENT]: Successfully sent notification to all users:', response)
    except Exception as e:
        print('Failed to send notification:', e)

if __name__ == "__main__":
    # Notification details
    message_title = "Critical Alert"
    message_body = "This is a critical alert message for all users."

    # Sending notification to all users
    send_notification_to_all(message_title, message_body)

```

Here are a few things to keep in mind while triggering the script:

- The default Notification object leaves it empty.
- Put the title and body in the data object.
- Define a critical notification by using the flag `critical = true` on Android and `critical = 1` on iOS.

Here’s the full [demo code.](https://github.com/sohagmahin/critical_alert_demo) You can check it out. Thanks.

![Critical alert demo](https://github.com/user-attachments/assets/34adec36-27b3-4818-bea5-666aa7ba05a2)

## References

- [Critical Alerts Entitlement Request](https://stackoverflow.com/questions/66057840/ios-how-do-you-implement-critical-alerts-for-your-app-when-you-dont-have-an-en)
- [how to implement critical alerts in ios 12](https://www.tapcode.co/2018/10/17/how-to-implement-critical-alerts-in-ios-12/)

Thanks for reading. If you have any questions or suggestions, feel free to reach out to me on [LinkedIn](https://www.linkedin.com/in/sohagmahin/). I’d love to hear from you.
