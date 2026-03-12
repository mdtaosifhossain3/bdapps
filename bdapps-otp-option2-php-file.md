# BDApps OTP Integration — PHP (Option 2: Simple)

## 1. Send OTP (`sent.php`)

A simple script to request a subscription OTP from BDApps.

```php
<?php
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);

$user_mobile = $_POST['user_mobile'];
$user_mobile = "tel:88" . $user_mobile;

// Save subscriber ID for reference
file_put_contents("user_number_fn.txt", $user_mobile);

// Request data
$requestData = array(
    "applicationId" => "app id",
    "password" => "password",
    "subscriberId" => "$user_mobile",
    "applicationHash" => "abchdef",
    "applicationMetaData" => array(
        "client" => "MOBILEAPP",
        "device" => "Samsung S10",
        "os" => "android 8",
        "appCode" => "https://play.google.com/store/apps/details?id=lk.dialog.megarunlor"
    )
);

$requestJson = json_encode($requestData);
$url = "https://developer.bdapps.com/subscription/otp/request";

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
if ($responseJson === false) {
    echo "cURL error: " . curl_error($ch);
} else {
    $response = json_decode($responseJson, true);
    if ($response === null) {
        echo "Invalid JSON in response: " . $responseJson;
    } else {
        echo "Status code: " . $response["statusCode"] . "\n";
        echo "Status detail: " . $response["statusDetail"] . "\n";
        echo "Reference number: " . $response["referenceNo"] . "\n";
        echo "Version: " . $response["version"] . "\n";
    }
}

curl_close($ch);
?>
```

---

## 2. Verify OTP (`verify.php`)

A simple script to verify the OTP provided by the user.

```php
<?php
$otp = $_POST["otp"];
$referenceNo = $_POST["referenceNo"];

// Request data
$requestData = array(
    "applicationId" => "APP_ID",
    "password" => "APP_PASSWORD",
    "referenceNo" => "$referenceNo",
    "otp" => "$otp"
);

$requestJson = json_encode($requestData);
$url = "https://developer.bdapps.com/subscription/otp/verify";

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
if ($responseJson === false) {
    echo "cURL error: " . curl_error($ch);
} else {
    $response = json_decode($responseJson, true);
    if ($response === null) {
        echo "Invalid JSON in response: " . $responseJson;
    } else {
        echo "Status code: " . $response["statusCode"] . "\n";
        echo "Status detail: " . $response["statusDetail"] . "\n";
        echo "Subscriber ID: " . $response["subscriberId"] . "\n";
        echo "Subscription status: " . $response["subscriptionStatus"] . "\n";
        echo "Version: " . $response["version"] . "\n";
    }
}

curl_close($ch);
?>
```