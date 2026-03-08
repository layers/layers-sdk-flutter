# Layers Flutter SDK

The Layers Flutter SDK provides analytics, attribution, and monetization tracking for Flutter apps on iOS and Android. It features event tracking, screen tracking, user identification, App Tracking Transparency (ATT), deep link handling, clipboard-based deferred deep links, consent management, and lifecycle-aware event flushing.

## Requirements

- Flutter 3.10.0+
- Dart SDK 3.0.0+
- iOS 14.0+ / Android API 21+

## Installation

Add `layers_flutter` to your `pubspec.yaml`:

```yaml
dependencies:
  layers_flutter: ^1.0.0
```

Then run:

```bash
flutter pub get
```

## Quick Start

```dart
import 'package:layers_flutter/layers_flutter.dart';

// Initialize in your app's main function
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  await Layers.init(LayersConfig(
    appId: 'your-app-id',
    environment: LayersEnvironment.production,
  ));

  runApp(MyApp());
}

// Track events anywhere
Layers.track('button_tapped', properties: {'button_name': 'signup'});

// Track screen views
Layers.screen('Home');

// Identify users
Layers.identify('user_123');
```

## Configuration

### LayersConfig

| Parameter | Type | Default | Description |
|---|---|---|---|
| `appId` | `String` | *required* | Your Layers application identifier. Must not be empty. |
| `environment` | `LayersEnvironment` | `.production` | Deployment environment. |
| `baseUrl` | `String?` | `null` | Custom base URL for the ingest API. Uses the production endpoint when `null`. |
| `enableDebug` | `bool` | `false` | Enable verbose debug logging via `dart:developer`. |
| `flushThreshold` | `int?` | `null` | Number of queued events that triggers an automatic flush. |
| `flushIntervalMs` | `int?` | `30000` | How often (in ms) the event queue is flushed automatically. |
| `maxQueueSize` | `int?` | `null` | Maximum events in the queue before dropping the oldest. |
| `autoTrackAppOpen` | `bool` | `true` | Whether to fire an `app_open` event during init. |
| `autoTrackDeepLinks` | `bool` | `true` | Whether to auto-track `deep_link_opened` events when a deep link is received. |

```dart
await Layers.init(LayersConfig(
  appId: 'your-app-id',
  environment: LayersEnvironment.development,
  enableDebug: true,
  flushIntervalMs: 15000,
  flushThreshold: 10,
  autoTrackAppOpen: true,
  autoTrackDeepLinks: true,
));
```

### LayersEnvironment

```dart
enum LayersEnvironment {
  development,
  staging,
  production,
}
```

## Core API

The SDK is accessed through static methods on the `Layers` class.

### Initialization

```dart
static Future<void> init(LayersConfig config)
```

Loads the native library, resolves the persistence path, collects device info, fetches remote config, reads clipboard attribution (iOS, if enabled), and fires an `app_open` event (unless `autoTrackAppOpen` is `false`). Also sets up deep link auto-tracking.

### Event Tracking

```dart
static void track(String eventName, {Map<String, dynamic>? properties})
```

```dart
Layers.track('purchase_completed', properties: {
  'product_id': 'sku_123',
  'price': 9.99,
  'currency': 'USD',
});
```

### Screen Tracking

```dart
static void screen(String screenName, {Map<String, dynamic>? properties})
```

```dart
Layers.screen('ProductDetail', properties: {'product_id': 'sku_123'});
```

### User Identity

```dart
// Identify the current user
static void identify(String userId)

// Set user-level properties
static void setUserProperties(Map<String, dynamic> properties)
```

```dart
// After login
Layers.identify('user_123');
Layers.setUserProperties({
  'email': 'user@example.com',
  'plan': 'premium',
});
```

### Consent Management

```dart
static void setConsent(ConsentSettings consent)
```

```dart
class ConsentSettings {
  final bool? analytics;    // null = not determined
  final bool? advertising;  // null = not determined
}
```

```dart
// User accepts all tracking
Layers.setConsent(ConsentSettings(analytics: true, advertising: true));

// User opts out of advertising only
Layers.setConsent(ConsentSettings(analytics: true, advertising: false));
```

### Flush, Reset & Shutdown

```dart
// Flush queued events to the server
static Future<void> flush()

// Reset user state for logout flows (flushes first, then clears identity)
static Future<void> reset()

// Shut down the SDK and persist remaining events
static void shutdown()
```

### Read-Only Properties

```dart
// Whether the SDK has been initialized
static bool get isInitialized

// The current session ID, or null
static String? get sessionId

// Number of events in the queue (0 if not initialized)
static int get queueDepth
```

### Debug

```dart
static void debugPrintState()
```

Prints the current SDK state to the console and `dart:developer`, including initialization status, session ID, queue depth, user ID, consent state, environment, and configuration.

### Error Handling

```dart
// Set an error callback (receives error messages from the native core)
static void Function(String error)? onError;
```

```dart
Layers.onError = (error) {
  print('Layers error: $error');
  // Forward to your crash reporting service
};
```

If `onError` is not set, errors are logged via `dart:developer`. Unrecoverable errors (e.g. queue full) throw `LayersQueueFullException` when no `onError` is registered.

## App Tracking Transparency (ATT)

The ATT module is available via the `ATT` class. It wraps the native iOS ATTrackingManager and returns safe defaults on Android.

### ATT API

```dart
// Check if ATT is available on this device (iOS 14.5+)
static Future<bool> isAvailable()

// Get the current tracking authorization status
static Future<ATTStatus> getStatus()

// Request tracking authorization (shows the system dialog on iOS)
static Future<ATTStatus> requestAuthorization()

// Get IDFA (null if not authorized or not iOS)
static Future<String?> getAdvertisingId()

// Get IDFV (always available on iOS)
static Future<String?> getVendorId()

// Whether the user has already been prompted
static Future<bool> hasBeenPrompted()
```

### Convenience Method on Layers

```dart
// Request ATT and automatically update device info + consent
static Future<ATTStatus> requestTrackingPermission()
```

This method:
1. Requests ATT authorization
2. Collects IDFA (if authorized) and IDFV
3. Updates device context with the identifiers and ATT status
4. Sets advertising consent based on the result

```dart
final status = await Layers.requestTrackingPermission();
if (status == ATTStatus.authorized) {
  print('User authorized tracking');
}
```

### ATTStatus

```dart
enum ATTStatus {
  notDetermined,
  restricted,
  denied,
  authorized,
}
```

> **Important**: Add `NSUserTrackingUsageDescription` to your `ios/Runner/Info.plist`.

## Deep Links

The deep links module is available via the `DeepLinks` class.

### Deep Links API

```dart
// Get the initial deep link that opened the app (cold start)
static Future<DeepLinkData?> getInitialLink()

// Add a listener for incoming deep links (warm start)
// Returns an unsubscribe function
static VoidCallback addListener(DeepLinkCallback callback)
```

### Auto-Tracking

When `autoTrackDeepLinks` is `true` (default), the SDK automatically tracks a `deep_link_opened` event for both initial and subsequent deep links. The event includes:
- `url` -- full URL
- `scheme`, `host`, `path` -- URL components
- All query parameters as flat top-level properties (UTM params, click IDs, etc.)

### Manual Deep Link Handling

```dart
final unsubscribe = DeepLinks.addListener((data) {
  print('Deep link received: ${data.url}');
  print('Host: ${data.host}');
  print('Path: ${data.path}');

  // Access query params
  final utmSource = data.queryParams['utm_source'];
  final clickId = data.queryParams['fbclid'];
});

// Later: unsubscribe()
```

### DeepLinkData

```dart
class DeepLinkData {
  final String url;
  final String? scheme;
  final String? host;
  final String? path;
  final Map<String, String> queryParams;
}
```

### Native Setup

**iOS**: Add URL schemes to `ios/Runner/Info.plist` and configure associated domains for Universal Links in your entitlements.

**Android**: Add intent filters to `android/app/src/main/AndroidManifest.xml`:

```xml
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="myapp" android:host="open" />
</intent-filter>
```

## Clipboard Attribution (iOS Deferred Deep Links)

On iOS, when a user clicks an ad, the landing page may copy a click URL to the clipboard. On first launch, the SDK reads the clipboard for a Layers attribution URL and includes it in the `app_open` event properties as `clipboard_attribution_url` and `clipboard_click_id`.

This feature is controlled by the server's remote config (`clipboard_attribution_enabled`). On iOS 16+, the system shows a paste consent dialog. No additional setup is required.

## Automatic Behaviors

- **app_open event**: Tracked on init with clipboard attribution (iOS, if enabled).
- **deep_link_opened event**: Tracked automatically for all incoming deep links (configurable).
- **Background flush**: Events are flushed when the app lifecycle state changes to paused or detached.
- **Periodic flush**: Events are flushed on a timer (configurable via `flushIntervalMs`).
- **Remote config**: Server-driven configuration is fetched during init.
- **Device context**: Platform, OS version, device model, locale, screen size, and timezone are collected automatically.
- **Event persistence**: Events are persisted to the Application Support directory and rehydrated on restart.
- **Retry with backoff**: Failed HTTP requests are retried on subsequent flush cycles.

## Navigator Observer for Screen Tracking

For automatic screen tracking with Flutter's Navigator:

```dart
class LayersNavigatorObserver extends NavigatorObserver {
  @override
  void didPush(Route<dynamic> route, Route<dynamic>? previousRoute) {
    if (route.settings.name != null) {
      Layers.screen(route.settings.name!);
    }
  }
}

// In your MaterialApp:
MaterialApp(
  navigatorObservers: [LayersNavigatorObserver()],
  // ...
)
```

## Testing

The SDK provides test helpers:

```dart
// Inject a mock bindings implementation
@visibleForTesting
static void enableTestMode(LayersBindings bindings)

// Reset all SDK state
@visibleForTesting
static void resetForTesting()
```

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:layers_flutter/layers_flutter.dart';

void main() {
  setUp(() {
    Layers.enableTestMode(MockLayersBindings());
  });

  tearDown(() {
    Layers.resetForTesting();
  });

  test('tracks event', () {
    Layers.track('test_event', properties: {'key': 'value'});
    // Assert against your mock
  });
}
```
