# BDApps OTP Integration — Option 1

## Overview

This document covers the BDApps OTP-based subscription flow using `flutter_riverpod`. It includes the auth state model, the Riverpod notifier, and the (commented-out) UI screens.

---

## Dependencies

```yaml
flutter_riverpod: <latest>
http: <latest>
shared_preferences: <latest>
```

---

## Auth State & Provider

### `AuthState`

```dart
import 'dart:async';
import 'dart:convert';
import 'package:flutter_riverpod/legacy.dart';
import 'package:http/http.dart' as http;
import 'package:shared_preferences/shared_preferences.dart';

/// State class for Auth
class AuthState {
  final bool isLoading;
  final String errorMessage;
  final String mobileNumber;
  final bool isSubscribed;

  AuthState({
    this.isLoading = false,
    this.errorMessage = '',
    this.mobileNumber = '',
    this.isSubscribed = false,
  });

  AuthState copyWith({
    bool? isLoading,
    String? errorMessage,
    String? mobileNumber,
    bool? isSubscribed,
  }) {
    return AuthState(
      isLoading: isLoading ?? this.isLoading,
      errorMessage: errorMessage ?? this.errorMessage,
      mobileNumber: mobileNumber ?? this.mobileNumber,
      isSubscribed: isSubscribed ?? this.isSubscribed,
    );
  }
}
```

### Provider

```dart
/// Provider for AuthController
final authProvider = StateNotifierProvider<AuthNotifier, AuthState>((ref) {
  return AuthNotifier();
});
```

---

## AuthNotifier

### `sendOTP`

Sends an OTP to the given mobile number via the BDApps API.

```dart
class AuthNotifier extends StateNotifier<AuthState> {
  AuthNotifier() : super(AuthState());


  Future<bool> sendOTP(String mobile) async {
    state = state.copyWith(isLoading: true, errorMessage: '');

    try {
      final sharedPrefs = await SharedPreferences.getInstance();
      await sharedPrefs.setString("MOBILE_NUMBER", mobile);

      state = state.copyWith(mobileNumber: mobile);
      Map<String, String> data = {'user_mobile': mobile};

      final response = await http
          .post(
            Uri.parse('https://example.com/api/sent.php'),
            headers: {'Content-Type': 'application/x-www-form-urlencoded'},
            body: data,
          )
          .timeout(const Duration(seconds: 15));

      if (response.statusCode == 200) {
        final jsonResponse = json.decode(response.body);

        if (jsonResponse.containsKey('error')) {
          state = state.copyWith(
            isLoading: false,
            errorMessage: jsonResponse['error'],
          );
          return false;
        }

        final statusCode = jsonResponse['status_code'];
        if (statusCode == "S1000") {
          final referenceNo = jsonResponse['reference_no'];
          sharedPrefs.setString("REFERENCE_NUMBER", referenceNo);
          state = state.copyWith(isLoading: false);
          return true;
        } else if (statusCode == "E1351") {
          await sharedPrefs.setBool("IS_SUBSCRIBED", true);
          state = state.copyWith(isLoading: false, isSubscribed: true);
          return false;
        } else {
          state = state.copyWith(
            isLoading: false,
            errorMessage:
                jsonResponse['status_detail'] ?? 'An error occurred. Please try again.',
          );
          return false;
        }
      }

      state = state.copyWith(
        isLoading: false,
        errorMessage: 'Failed to send OTP. Try again.',
      );
      return false;
    } catch (e) {
      state = state.copyWith(
        isLoading: false,
        errorMessage: 'Network error. Check connection.',
      );
      return false;
    }
  }
```

### `verifyOTP`

Verifies the OTP entered by the user.

```dart
  Future<bool> verifyOTP(String otp) async {
    state = state.copyWith(isLoading: true, errorMessage: '');

    try {
      final sharedPrefs = await SharedPreferences.getInstance();
      final refNum = sharedPrefs.getString("REFERENCE_NUMBER");

      if (refNum == null) {
        state = state.copyWith(
          isLoading: false,
          errorMessage: 'Reference not found.',
        );
        return false;
      }

      Map<String, String> data = {"referenceNo": refNum, "otp": otp};
      final response = await http
          .post(
            Uri.parse('https://example.com/api/verify.php'),
            headers: {'Content-Type': 'application/x-www-form-urlencoded'},
            body: data,
          )
          .timeout(const Duration(seconds: 15));

      if (response.statusCode == 200) {
        final jsonResponse = json.decode(response.body);
        final status = jsonResponse['status'];

        if (status == 'success') {
          final data = jsonResponse['data'];
          final statusCode = data['status_code'];

          if (statusCode == 'S1000' || statusCode == 'E1854') {
            await sharedPrefs.setBool("IS_SUBSCRIBED", true);
            state = state.copyWith(isLoading: false, isSubscribed: true);
            return true;
          } else {
            state = state.copyWith(
              isLoading: false,
              errorMessage: data['status_detail'] ?? 'Verification failed',
            );
            return false;
          }
        } else {
          final message = jsonResponse['message'] ?? 'Enter a valid OTP';
          state = state.copyWith(isLoading: false, errorMessage: message);
          return false;
        }
      } else if (response.statusCode == 400) {
        final jsonResponse = json.decode(response.body);
        state = state.copyWith(
          isLoading: false,
          errorMessage: jsonResponse['message'] ?? 'Invalid or expired OTP',
        );
        return false;
      } else {
        state = state.copyWith(
          isLoading: false,
          errorMessage: 'Server Error: ${response.statusCode}',
        );
        return false;
      }
    } catch (e) {
      state = state.copyWith(
        isLoading: false,
        errorMessage: 'Verification failed.',
      );
      return false;
    }
  }
```

### `unsubscribe`

Unsubscribes the user and clears local storage.

```dart
  Future<void> unsubscribe() async {
    state = state.copyWith(isLoading: true);
    final sharedPrefs = await SharedPreferences.getInstance();
    final mobile = sharedPrefs.getString("MOBILE_NUMBER") ?? "";

    try {
      final res = await http
          .post(
            Uri.parse('https://example.com/api/unsubcribe.php'),
            headers: {'Content-Type': 'application/json'},
            body: jsonEncode({
              "applicationId": "appid",
              "password": "app_password",
              "version": "1.0",
              "action": 0,
              "subscriberId": "tel:88$mobile",
            }),
          )
          .timeout(const Duration(seconds: 15));

      print(res.body);
      await sharedPrefs.setBool("IS_SUBSCRIBED", false);
      await sharedPrefs.clear();
      state = AuthState();
    } catch (e) {
      state = state.copyWith(isLoading: false);
      rethrow;
    }
  }
}
```
### `splash screen`

```dart
Future<void> _checkSubscriptionAndNavigate() async {
    final sharedPrefs = await SharedPreferences.getInstance();
    final isSubscribed = sharedPrefs.getBool("IS_SUBSCRIBED") ?? false;

    await Future.delayed(const Duration(seconds: 3));

    if (mounted) {
      Navigator.pushReplacement(
        context,
        MaterialPageRoute(
          builder: (context) => isSubscribed
              ? const HomeScreen()
              : const AmarAppsSubscriptionPage(),
        ),
      );
    }
  }
```

