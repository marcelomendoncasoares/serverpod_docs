# Custom Token Manager

When implementing custom authentication, you may need to create a custom token manager (also called an auth key provider) to handle how authentication tokens are provided to the client. This is useful when you have custom token formats, refresh logic, or authentication schemes that aren't covered by the built-in token managers.

## Overview

A custom token manager implements the `ClientAuthKeyProvider` interface, which is responsible for providing authentication header values to include in requests to the server. The `FlutterAuthSessionManager` uses these token managers to handle authentication tokens for different authentication strategies.

## Creating a Simple Custom Token Manager

For simple token managers that don't need token refresh capabilities, implement the `ClientAuthKeyProvider` interface directly:

```dart
import 'package:serverpod_auth_core_client/serverpod_auth_core_client.dart';

/// A simple custom token manager that provides tokens as bearer tokens.
class CustomTokenManager implements ClientAuthKeyProvider {
  /// The function to get the authentication info of the user.
  final Future<AuthSuccess?> Function() getAuthInfo;

  /// Creates a new [CustomTokenManager].
  CustomTokenManager({
    required this.getAuthInfo,
  });

  @override
  Future<String?> get authHeaderValue async {
    final currentAuth = await getAuthInfo();
    if (currentAuth == null) return null;
    
    // Wrap the token as a bearer token
    return wrapAsBearerAuthHeaderValue(currentAuth.token);
    
    // Or use basic auth if preferred:
    // return wrapAsBasicAuthHeaderValue(currentAuth.token);
  }
}
```

This simple token manager:
- Retrieves the current authentication info using the provided `getAuthInfo` callback
- Returns `null` if no user is authenticated
- Wraps the token as a bearer token for HTTP requests

## Creating a Token Manager with Refresh Capability

For token managers that need to refresh tokens before they expire, implement the `RefresherClientAuthKeyProvider` interface or extend `MutexRefresherClientAuthKeyProvider`:

```dart
import 'package:serverpod_auth_core_client/serverpod_auth_core_client.dart';

/// A custom token manager with refresh capability.
class CustomRefreshingTokenManager extends MutexRefresherClientAuthKeyProvider {
  /// Creates a new [CustomRefreshingTokenManager].
  CustomRefreshingTokenManager({
    required Future<AuthSuccess?> Function() getAuthInfo,
    required Future<void> Function(AuthSuccess authSuccess) onRefreshAuthInfo,
    required YourCustomRefreshEndpoint refreshEndpoint,
    Duration refreshTokenBefore = const Duration(seconds: 30),
  }) : super(
         _CustomRefreshingTokenManagerDelegate(
           getAuthInfo: getAuthInfo,
           onRefreshAuthInfo: onRefreshAuthInfo,
           refreshEndpoint: refreshEndpoint,
           refreshTokenBefore: refreshTokenBefore,
         ),
       );
}

class _CustomRefreshingTokenManagerDelegate implements RefresherClientAuthKeyProvider {
  final Future<AuthSuccess?> Function() getAuthInfo;
  final Future<void> Function(AuthSuccess authSuccess) onRefreshAuthInfo;
  final YourCustomRefreshEndpoint refreshEndpoint;
  final Duration refreshTokenBefore;

  _CustomRefreshingTokenManagerDelegate({
    required this.getAuthInfo,
    required this.onRefreshAuthInfo,
    required this.refreshEndpoint,
    required this.refreshTokenBefore,
  });

  @override
  Future<String?> get authHeaderValue async {
    final currentAuth = await getAuthInfo();
    if (currentAuth == null) return null;
    return wrapAsBearerAuthHeaderValue(currentAuth.token);
  }

  @override
  Future<RefreshAuthKeyResult> refreshAuthKey({bool force = false}) async {
    final currentAuthInfo = await getAuthInfo();
    final currentExpiresAt = currentAuthInfo?.tokenExpiresAt;
    final refreshToken = currentAuthInfo?.refreshToken;

    // Skip refresh if token is not expiring soon (unless forced)
    if (!force && 
        (currentExpiresAt == null || 
         !_isExpiringSoon(currentExpiresAt, refreshTokenBefore) ||
         refreshToken == null)) {
      return RefreshAuthKeyResult.skipped;
    }

    try {
      // Call your custom refresh endpoint
      final authSuccess = await refreshEndpoint.refreshToken(
        refreshToken: refreshToken!,
      );
      
      // Update the stored auth info
      await onRefreshAuthInfo(authSuccess);
      return RefreshAuthKeyResult.success;
    } catch (e) {
      // Handle specific error types
      if (e is RefreshTokenExpiredException ||
          e is RefreshTokenInvalidException) {
        return RefreshAuthKeyResult.failedUnauthorized;
      }
      return RefreshAuthKeyResult.failedOther;
    }
  }

  bool _isExpiringSoon(DateTime expiresAt, Duration before) {
    return DateTime.now().toUtc().add(before).isAfter(expiresAt);
  }
}
```

This refreshing token manager:
- Extends `MutexRefresherClientAuthKeyProvider` to prevent concurrent refresh attempts
- Checks if the token is expiring soon before refreshing
- Calls a custom refresh endpoint when needed
- Handles refresh errors appropriately
- Updates the stored authentication info after successful refresh

## Using Custom Token Managers with FlutterAuthSessionManager

To use your custom token manager with `FlutterAuthSessionManager`, pass it via the `authKeyProviderDelegates` parameter:

### Setup

Add the `serverpod_auth_core_flutter` package to your app's `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  serverpod_auth_core_flutter: 3.x.x
  serverpod_flutter: 3.x.x
  your_client:
    path: ../your_client
```

Create and configure the `FlutterAuthSessionManager` with your custom token manager:

```dart
import 'package:flutter/material.dart';
import 'package:serverpod_flutter/serverpod_flutter.dart';
import 'package:serverpod_auth_core_flutter/serverpod_auth_core_flutter.dart';
import 'package:your_client/your_client.dart';

late Client client;

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  const serverUrl = 'http://localhost:8080/';

  // Create the FlutterAuthSessionManager with your custom token manager
  final authSessionManager = FlutterAuthSessionManager(
    authKeyProviderDelegates: {
      'custom': CustomTokenManager(
        getAuthInfo: () async => authSessionManager.authInfo,
      ),
      // You can register multiple token managers for different auth strategies
      'custom-refreshing': CustomRefreshingTokenManager(
        getAuthInfo: () async => authSessionManager.authInfo,
        onRefreshAuthInfo: authSessionManager.updateSignedInUser,
        refreshEndpoint: client.yourCustomRefreshEndpoint,
      ),
    },
  );

  // Create the client with the auth session manager
  client = Client(serverUrl)
    ..connectivityMonitor = FlutterConnectivityMonitor()
    ..authSessionManager = authSessionManager;

  // Initialize authentication (restores session from storage and validates)
  await client.auth.initialize();

  runApp(MyApp());
}
```

The `FlutterAuthSessionManager` will automatically use the appropriate token manager based on the `authStrategy` stored in the `AuthSuccess` object when you authenticate.

### Storing Authentication Tokens

After successfully authenticating with your custom endpoint, store the authentication information using `updateSignedInUser`:

```dart
// After successful authentication with your custom endpoint
final loginResponse = await client.myCustomAuthEndpoint.login(
  username: username,
  password: password,
);

if (loginResponse != null && loginResponse.token != null) {
  // Create AuthSuccess object with your token and auth strategy
  final authSuccess = AuthSuccess(
    authKey: loginResponse.token!,
    authStrategy: 'custom', // Must match the key in authKeyProviderDelegates
    userId: loginResponse.userId,
    // Optional: Include refresh token if using refreshing token manager
    refreshToken: loginResponse.refreshToken,
    // Optional: Include token expiration if using refreshing token manager
    tokenExpiresAt: loginResponse.tokenExpiresAt,
  );

  // Store the authentication info
  await client.auth.updateSignedInUser(authSuccess);
}
```

:::important
The `authStrategy` value in `AuthSuccess` must match one of the keys in the `authKeyProviderDelegates` map. This is how `FlutterAuthSessionManager` knows which token manager to use.
:::

## Reactive UI Updates

The `FlutterAuthSessionManager` provides `authInfoListenable`, a `ValueListenable` that you can use to reactively update your UI when authentication state changes:

### Using ValueListenableBuilder (Recommended)

```dart
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: ValueListenableBuilder<AuthSuccess?>(
        valueListenable: client.auth.authInfoListenable,
        builder: (context, authInfo, child) {
          final isAuthenticated = authInfo != null;
          return isAuthenticated ? HomeScreen() : LoginScreen();
        },
      ),
    );
  }
}
```

### Using Listeners

```dart
class MyApp extends StatefulWidget {
  @override
  State<MyApp> createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  @override
  void initState() {
    super.initState();
    // Listen to authentication state changes
    client.auth.authInfoListenable.addListener(_onAuthChanged);
  }

  @override
  void dispose() {
    client.auth.authInfoListenable.removeListener(_onAuthChanged);
    super.dispose();
  }

  void _onAuthChanged() {
    setState(() {
      // UI will rebuild when auth state changes
    });
  }

  @override
  Widget build(BuildContext context) {
    final isAuthenticated = client.auth.isAuthenticated;
    
    return MaterialApp(
      home: isAuthenticated ? HomeScreen() : LoginScreen(),
    );
  }
}
```

## Session Management

The `FlutterAuthSessionManager` provides several methods for managing user sessions:

- `initialize()` - Restores session from storage and validates with the server
- `restore()` - Restores session from storage without server validation
- `validateAuthentication()` - Validates the current session with the server
- `signOutDevice()` - Signs out the user from the current device
- `signOutAllDevices()` - Signs out the user from all devices

```dart
// Sign out the current device
await client.auth.signOutDevice();

// Sign out from all devices
await client.auth.signOutAllDevices();
```

## Authentication Schemes

By default, Serverpod uses bearer tokens for authentication. Your custom token manager can return different authentication schemes:

- **Bearer tokens**: Use `wrapAsBearerAuthHeaderValue(token)` - recommended for most cases
- **Basic auth**: Use `wrapAsBasicAuthHeaderValue(token)` - for legacy compatibility

The token manager's `authHeaderValue` getter should return a properly formatted HTTP Authorization header value compliant with RFC 9110.

## Example: Complete Custom Authentication Flow

Here's a complete example showing how to implement custom authentication with a custom token manager:

```dart
// 1. Define your custom token manager
class MyCustomTokenManager implements ClientAuthKeyProvider {
  final Future<AuthSuccess?> Function() getAuthInfo;

  MyCustomTokenManager({required this.getAuthInfo});

  @override
  Future<String?> get authHeaderValue async {
    final auth = await getAuthInfo();
    if (auth == null) return null;
    return wrapAsBearerAuthHeaderValue(auth.token);
  }
}

// 2. Set up the client with FlutterAuthSessionManager
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  final authSessionManager = FlutterAuthSessionManager(
    authKeyProviderDelegates: {
      'my-custom-auth': MyCustomTokenManager(
        getAuthInfo: () async => authSessionManager.authInfo,
      ),
    },
  );

  client = Client('http://localhost:8080/')
    ..connectivityMonitor = FlutterConnectivityMonitor()
    ..authSessionManager = authSessionManager;

  await client.auth.initialize();
  runApp(MyApp());
}

// 3. Authenticate and store the token
Future<void> login(String username, String password) async {
  final response = await client.myAuthEndpoint.login(
    username: username,
    password: password,
  );

  if (response?.token != null) {
    await client.auth.updateSignedInUser(
      AuthSuccess(
        authKey: response!.token!,
        authStrategy: 'my-custom-auth',
        userId: response.userId,
      ),
    );
  }
}

// 4. Use reactive UI updates
class AuthWrapper extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ValueListenableBuilder<AuthSuccess?>(
      valueListenable: client.auth.authInfoListenable,
      builder: (context, authInfo, child) {
        return authInfo != null ? AuthenticatedApp() : LoginScreen();
      },
    );
  }
}
```

This example demonstrates the complete flow from creating a custom token manager to using it in a Flutter application with reactive UI updates.
