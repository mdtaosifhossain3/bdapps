# BDApps OTP Integration — PHP (Option 1: Advanced with Logging)

## 1. Send OTP (`sent.php`)

This script handles the subscription OTP request, logs API calls, and manages user creation in the database.

```php
<?php
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);

// Set Bangladesh timezone
date_default_timezone_set('Asia/Dhaka');

// BDApps API configuration
define('APP_ID', 'app id');
define('APP_PASSWORD', 'app password');
define('APP_HASH', 'abchdef');

// Get mobile from POST
$userMobile = $_POST['user_mobile'] ?? '';
if (empty($userMobile)) {
    die(json_encode(['error' => 'Mobile number is required']));
}

// Format subscriber ID
$subscriberId = "tel:88" . $userMobile;
$mobileNumber = "88" . $userMobile;

// Save to file
file_put_contents("user_number_fn.txt", $subscriberId);

// Prepare request data
$requestData = array(
    "applicationId" => APP_ID,
    "password" => APP_PASSWORD,
    "subscriberId" => $subscriberId,
    "applicationHash" => APP_HASH,
    "applicationMetaData" => array(
        "client" => "MOBILEAPP",
        "device" => "Samsung S10",
        "os" => "android 8",
        "appCode" => "https://play.google.com/store/apps/details?id=lk.dialog.megarunlor"
    )
);

$requestJson = json_encode($requestData);
$url = "https://developer.bdapps.com/subscription/otp/request";

$startTime = microtime(true);

// cURL setup
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, $url);
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, $requestJson);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_HTTPHEADER, array(
    "Content-Type: application/json",
    "Content-Length: " . strlen($requestJson)
));

$responseJson = curl_exec($ch);
$httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
$responseTime = round((microtime(true) - $startTime) * 1000);

if ($responseJson === false) {
    $error = curl_error($ch);
    echo json_encode(['error' => 'cURL error: ' . $error]);
    curl_close($ch);
    exit;
}

curl_close($ch);

// Parse response
$response = json_decode($responseJson, true);

if ($response === null) {
    echo json_encode(['error' => 'Invalid JSON in response', 'raw_response' => $responseJson]);
    exit;
}

// Return response
header('Content-Type: application/json');
echo json_encode([
    'status_code' => $response["statusCode"] ?? null,
    'status_detail' => $response["statusDetail"] ?? null,
    'reference_no' => $response["referenceNo"] ?? $response["reference_no"] ?? null,
    'version' => $response["version"] ?? null,
    'subscriber_id' => $subscriberId,
    'response_time_ms' => $responseTime,
    'http_code' => $httpCode
]);
?>
```

---

## 2. Verify OTP (`verify.php`)

This script verifies the OTP with BDApps and activates the user in the database upon success.

```php
<?php
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);

// Set Bangladesh timezone
date_default_timezone_set('Asia/Dhaka');

// BDApps API configuration
define('APP_ID', 'app id');
define('APP_PASSWORD', 'app password');

// Get POST parameters
$otp = $_POST["otp"] ?? '';
$referenceNo = $_POST["referenceNo"] ?? '';

// Validate inputs
if (empty($otp) || empty($referenceNo)) {
    header('Content-Type: application/json');
    echo json_encode([
        'status' => 'error',
        'message' => 'OTP and reference number are required'
    ]);
    exit;
}

// Prepare BDApps API request
$requestData = array(
    "applicationId" => "APP_ID",
    "password" => "APP_PASSWORD",
    "referenceNo" => $referenceNo,
    "otp" => $otp
);

$requestJson = json_encode($requestData);
$url = "https://developer.bdapps.com/subscription/otp/verify";

// cURL setup
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, $url);
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, $requestJson);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_HTTPHEADER, array(
    "Content-Type: application/json",
    "Content-Length: " . strlen($requestJson)
));

$responseJson = curl_exec($ch);
$httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);

if ($responseJson === false) {
    $error = curl_error($ch);
    curl_close($ch);

    header('Content-Type: application/json');
    echo json_encode([
        'status' => 'error',
        'message' => 'cURL error: ' . $error
    ]);
    exit;
}

curl_close($ch);

$response = json_decode($responseJson, true);

if ($response === null) {
    header('Content-Type: application/json');
    echo json_encode([
        'status' => 'error',
        'message' => 'Invalid JSON in response',
        'raw_response' => $responseJson
    ]);
    exit;
}

$isVerified = ($response["statusCode"] === "S1000");
$subscriptionStatus = $response["subscriptionStatus"] ?? null;

header('Content-Type: application/json');

if ($isVerified) {
    echo json_encode([
        'status' => 'success',
        'message' => 'OTP verified successfully',
        'data' => [
            'reference_no' => $referenceNo,
            'subscriber_id' => $response["subscriberId"] ?? null,
            'subscription_status' => $subscriptionStatus,
            'status_code' => $response["statusCode"],
            'status_detail' => $response["statusDetail"],
            'version' => $response["version"] ?? null
        ]
    ]);
} else {
    echo json_encode([
        'status' => 'error',
        'message' => 'OTP verification failed',
        'data' => [
            'reference_no' => $referenceNo,
            'status_code' => $response["statusCode"] ?? null,
            'status_detail' => $response["statusDetail"] ?? null,
            'version' => $response["version"] ?? null
        ]
    ]);
}
?>
```

## 3. Unsubscribe (`unsubscribe.php`)

This script unsubscribes the user from the service.

```php
<?php

// Read raw JSON input
$input = file_get_contents("php://input");
$data  = json_decode($input, true);

// Extract JSON values
$applicationId = $data['applicationId'] ?? null;
$password      = $data['password'] ?? null;
$version       = $data['version'] ?? "1.0";
$action        = $data['action'] ?? null;
$subscriberId  = $data['subscriberId'] ?? null;

// Validate required fields
if (!$applicationId || !$password || $action === null || !$subscriberId) {
    echo json_encode([
        "error" => "applicationId, password, action and subscriberId are required"
    ]);
    exit;
}

// BDApps subscription API (HTTPS required)
$url = "https://developer.bdapps.com/subscription/send";

// Build request body
$body = json_encode([
    "applicationId" => $applicationId,
    "password"      => $password,
    "version"       => $version,
    "action"        => (int)$action,
    "subscriberId"  => $subscriberId
]);

// cURL request
$ch = curl_init($url);
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, $body);
curl_setopt($ch, CURLOPT_HTTPHEADER, ["Content-Type: application/json"]);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

$response = curl_exec($ch);
curl_close($ch);

// Return BDApps response
echo $response;
?>
```