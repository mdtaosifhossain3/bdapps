# BDApps OTP Integration — Option 2 (GetX)

## Overview

This option uses **GetX** (`get`) for state management instead of Riverpod. The `AuthController` handles mobile number validation, OTP send/verify, and unsubscription.

---

## Dependencies

```yaml
get: <latest>
http: <latest>
shared_preferences: <latest>
```

---

## AuthController

### Setup & State

```dart
import 'dart:async';
import 'dart:io';

import 'package:get/get.dart';
import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;
import 'package:salatapp/pages/main_nav_page.dart';
import 'package:shared_preferences/shared_preferences.dart';
import '../services/auth_service.dart';

/// Controller for authentication flow
/// Manages mobile number input, OTP verification, and subscription
class AuthController extends GetxController {
  final mobileController = TextEditingController();

  // Observable states
  final isLoading = false.obs;
  final errorMessage = ''.obs;
  final mobileNumber = ''.obs;

  @override
  void onClose() {
    mobileController.dispose();
    super.onClose();
  }
}
```

### `validateMobileNumber`

```dart
  String? validateMobileNumber(String? value) {
    if (value == null || value.isEmpty) {
      return 'Please enter your mobile number';
    }

    String cleanNumber = value.replaceAll(RegExp(r'[^\d]'), '');

    if (cleanNumber.length != 11) {
      return 'Mobile number must be 11 digits';
    }

    if (!AuthService.isValidRobiAirtelNumber(cleanNumber)) {
      return 'Please enter a valid Robi/Airtel number';
    }

    return null;
  }
```

### `extractValue` (Helper)

```dart
  String extractValue(String body, String key) {
    final startIndex = body.indexOf(key);
    if (startIndex != -1) {
      final substring = body.substring(startIndex);
      final endIndex = substring.indexOf('\n');
      if (endIndex != -1) {
        return substring.substring(key.length, endIndex).trim();
      }
      return substring.substring(key.length).trim();
    }
    return '';
  }
```

### `sendOTP`

```dart
  Future<bool> sendOTP() async {
    errorMessage.value = '';

    String? validationError = validateMobileNumber(mobileController.text);
    if (validationError != null) {
      errorMessage.value = validationError;
      return false;
    }

    isLoading.value = true;

    try {
      mobileNumber.value = AuthService.formatMobileNumber(mobileController.text);
      SharedPreferences sharedPreferences = await SharedPreferences.getInstance();
      sharedPreferences.setString("MOBILE_NUMBER", mobileNumber.value);

      Map<String, String> data = {'user_mobile': mobileNumber.value};

      final response = await http
          .post(
            Uri.parse('https://flicksize.com/v1/fsize_sent_otp.php'),
            body: data,
          )
          .timeout(const Duration(seconds: 15));

      if (response.statusCode == 200) {
        var body = response.body;
        final statusCode = extractValue(body, 'Status code').trim();
        final result = statusCode.replaceAll(":", "").trim();

        if (result == "S1000") {
          final ref = extractValue(body, 'Reference number');
          final refResult = ref.replaceAll(":", "").trim();
          sharedPreferences.setString("REFERENCE_NUMBER", refResult);
          return true;
        } else if (result == "E1351") {
          Get.offAll(() => const MainNavPage());
          return false;
        } else {
          errorMessage.value = 'An error occurred. Please try again.';
          return false;
        }
      } else {
        errorMessage.value = 'An error occurred. Please try again.';
        return false;
      }
    } on SocketException {
      errorMessage.value = 'Network Issue: Check your internet connection';
      return false;
    } on TimeoutException {
      errorMessage.value = 'Connection timed out. Please try again.';
      return false;
    } catch (e) {
      errorMessage.value = 'An error occurred. Please try again.';
      return false;
    } finally {
      isLoading.value = false;
    }
  }
```

### `verifyOTP`

```dart
  Future<bool> verifyOTP(String otp) async {
    errorMessage.value = '';

    if (otp.isEmpty) {
      errorMessage.value = 'Please enter OTP';
      return false;
    }

    if (otp.length != 6) {
      errorMessage.value = 'OTP must be 6 digits';
      return false;
    }

    isLoading.value = true;

    try {
      SharedPreferences sharedPreferences = await SharedPreferences.getInstance();
      final referenceNumber = sharedPreferences.getString("REFERENCE_NUMBER");

      if (referenceNumber == null) {
        errorMessage.value = 'Reference number not found';
        return false;
      }

      Map<String, String> data = {"referenceNo": referenceNumber, "otp": otp};
      final response = await http
          .post(
            Uri.parse('https://flicksize.com/v1/fsize_verify_otp.php'),
            body: data,
          )
          .timeout(const Duration(seconds: 15));

      if (response.statusCode == 200) {
        var body = response.body;
        final statusCode = extractValue(body, 'Status code').trim();
        final result = statusCode.replaceAll(":", "").trim();

        if (result == "S1000") {
          await sharedPreferences.setString("USER_PHONE", mobileNumber.value);
          await sharedPreferences.setBool("IS_SUBSCRIBED", true);
          return true;
        } else {
          errorMessage.value = 'Enter a valid OTP';
          return false;
        }
      } else {
        errorMessage.value = 'Server Error: ${response.statusCode}';
        return false;
      }
    } on SocketException {
      errorMessage.value = 'Network Issue: Check your internet connection';
      return false;
    } on TimeoutException {
      errorMessage.value = 'Connection timed out. Please try again.';
      return false;
    } catch (e) {
      errorMessage.value = 'An error occurred. Please try again.';
      return false;
    } finally {
      isLoading.value = false;
    }
  }
```

### `clearForm`

```dart
  void clearForm() {
    mobileController.clear();
    mobileNumber.value = '';
    errorMessage.value = '';
  }
```

---

## Unsubscribe

```dart
Future<void> unsubscribe() async {
  SharedPreferences sharedPreferences = await SharedPreferences.getInstance();
  String mobile = sharedPreferences.getString("MOBILE_NUMBER") ?? "";

  try {
    await http
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

    Get.offAll(() => const AuthPage());
    Get.snackbar("Success", "Successfully unsubscribed");
    sharedPreferences.setBool("IS_SUBSCRIBED", false);
    sharedPreferences.clear();
  } on SocketException {
    Get.snackbar("Error", "No Internet");
  } on TimeoutException {
    Get.snackbar("Error", "Time Out");
  } catch (e) {
    Get.snackbar("Error", "Error: ${e.toString()}");
  }
}
```

---

## Splash Screen Navigation

```dart
Future<void> _checkAuthAndNavigate() async {
  await Future.delayed(const Duration(seconds: 3));

  SharedPreferences prefs = await SharedPreferences.getInstance();
  bool isSubscribed = prefs.getBool("IS_SUBSCRIBED") ?? false;

  if (isSubscribed) {
    Get.offAll(() => const MainNavPage());
  } else {
    Get.offAll(() => const AuthPage());
  }
}
```
