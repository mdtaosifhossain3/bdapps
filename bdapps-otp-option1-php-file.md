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

// Database configuration
define('DB_HOST', 'localhost');
define('DB_USER', 'user');
define('DB_PASS', 'pass');
define('DB_NAME', 'name');

// BDApps API configuration
define('APP_ID', 'APP_ID');
define('APP_PASSWORD', 'APP_PASSWORD');
define('APP_HASH', 'APP_HASH');

class OTPManager {
    private $conn;
    private $logId;
    
    public function __construct() {
        $this->connectDB();
    }
    
    private function connectDB() {
        $this->conn = new mysqli(DB_HOST, DB_USER, DB_PASS, DB_NAME);
        if ($this->conn->connect_error) {
            die("Database connection failed: " . $this->conn->connect_error);
        }
        $this->conn->set_charset("utf8mb4");
    }
    
    public function logApiStart($apiType, $endpoint, $method) {
        $stmt = $this->conn->prepare("INSERT INTO api_logs (api_type, endpoint, request_method, created_at) VALUES (?, ?, ?, NOW())");
        $stmt->bind_param("sss", $apiType, $endpoint, $method);
        $stmt->execute();
        $this->logId = $stmt->insert_id;
        $stmt->close();
        return $this->logId;
    }
    
    public function logApiComplete($responseBody, $responseTime, $httpCode, $error = null) {
        $stmt = $this->conn->prepare("UPDATE api_logs SET response_body = ?, response_time_ms = ?, http_code = ?, error_message = ? WHERE id = ?");
        $stmt->bind_param("siisi", $responseBody, $responseTime, $httpCode, $error, $this->logId);
        $stmt->execute();
        $stmt->close();
    }
    
    public function getOrCreateUser($mobileNumber, $subscriberId) {
        $stmt = $this->conn->prepare("SELECT id, status FROM users WHERE mobile_number = ?");
        $stmt->bind_param("s", $mobileNumber);
        $stmt->execute();
        $stmt->bind_result($userId, $currentStatus);
        
        if ($stmt->fetch()) {
            $stmt->close();
            $this->conn->query("UPDATE users SET last_login = NOW() WHERE id = " . intval($userId));
            return $userId;
        } else {
            $stmt->close();
            $stmt = $this->conn->prepare("INSERT INTO users (mobile_number, subscriber_id, status) VALUES (?, ?, 'pending')");
            $stmt->bind_param("ss", $mobileNumber, $subscriberId);
            $stmt->execute();
            $newUserId = $stmt->insert_id;
            $stmt->close();
            return $newUserId;
        }
    }
    
    public function saveOTPRequest($userId, $mobileNumber, $subscriberId, $requestData, $responseData) {
        $referenceNo = $responseData['referenceNo'] ?? $responseData['reference_no'] ?? null;
        $statusCode = $responseData['statusCode'] ?? null;
        $statusDetail = $responseData['statusDetail'] ?? null;
        $version = $responseData['version'] ?? null;
        $requestStatus = ($statusCode === 'S1000') ? 'success' : 'failed';
        $requestJson = json_encode($requestData);
        $responseJson = json_encode($responseData);
        
        $stmt = $this->conn->prepare("INSERT INTO otp_requests 
            (user_id, mobile_number, subscriber_id, reference_no, request_payload, response_payload, status_code, status_detail, version, request_status, expires_at) 
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, DATE_ADD(NOW(), INTERVAL 2 MINUTE))");
        
        $stmt->bind_param("isssssssss", 
            $userId, $mobileNumber, $subscriberId, $referenceNo, 
            $requestJson, $responseJson, $statusCode, $statusDetail, $version, $requestStatus
        );
        
        $stmt->execute();
        $otpRequestId = $stmt->insert_id;
        $stmt->close();
        
        return [
            'id' => $otpRequestId,
            'reference_no' => $referenceNo
        ];
    }
    
    public function getPendingOTP($mobileNumber, $referenceNo) {
        $stmt = $this->conn->prepare("SELECT id, user_id, mobile_number, subscriber_id, reference_no, request_payload, response_payload, status_code, status_detail, version, request_status, created_at, expires_at, verified_at FROM otp_requests 
            WHERE mobile_number = ? AND reference_no = ? AND request_status = 'success' 
            AND verified_at IS NULL AND expires_at > NOW() 
            ORDER BY created_at DESC LIMIT 1");
        $stmt->bind_param("ss", $mobileNumber, $referenceNo);
        $stmt->execute();
        
        $stmt->bind_result($id, $user_id, $mobile, $subscriber_id, $ref_no, $req_payload, $resp_payload, $status_code, $status_detail, $version, $req_status, $created_at, $expires_at, $verified_at);
        
        if ($stmt->fetch()) {
            $data = array(
                'id' => $id,
                'user_id' => $user_id,
                'mobile_number' => $mobile,
                'subscriber_id' => $subscriber_id,
                'reference_no' => $ref_no,
                'request_payload' => $req_payload,
                'response_payload' => $resp_payload,
                'status_code' => $status_code,
                'status_detail' => $status_detail,
                'version' => $version,
                'request_status' => $req_status,
                'created_at' => $created_at,
                'expires_at' => $expires_at,
                'verified_at' => $verified_at
            );
            $stmt->close();
            return $data;
        } else {
            $stmt->close();
            return null;
        }
    }
    
    public function saveOTPVerification($otpRequestId, $userId, $otpCode, $referenceNo, $requestData, $responseData, $isVerified) {
        $statusCode = $responseData['statusCode'] ?? null;
        $statusDetail = $responseData['statusDetail'] ?? null;
        $requestJson = json_encode($requestData);
        $responseJson = json_encode($responseData);
        $verifiedAt = $isVerified ? date('Y-m-d H:i:s') : null;
        
        $stmt = $this->conn->prepare("INSERT INTO otp_verifications 
            (otp_request_id, user_id, otp_code, reference_no, request_payload, response_payload, status_code, status_detail, is_verified, verified_at) 
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)");
        
        $stmt->bind_param("iissssssis", 
            $otpRequestId, $userId, $otpCode, $referenceNo, 
            $requestJson, $responseJson, $statusCode, $statusDetail, $isVerified, $verifiedAt
        );
        
        $stmt->execute();
        $verificationId = $stmt->insert_id;
        $stmt->close();
        
        if ($isVerified) {
            $this->conn->query("UPDATE otp_requests SET verified_at = NOW() WHERE id = " . intval($otpRequestId));
        }
        
        return $verificationId;
    }
    
    public function activateUser($userId) {
        $stmt = $this->conn->prepare("UPDATE users SET status = 'active' WHERE id = ?");
        $stmt->bind_param("i", $userId);
        $stmt->execute();
        $stmt->close();
    }
    
    public function close() {
        $this->conn->close();
    }
}

// Initialize OTP Manager
$otpManager = new OTPManager();

// Get mobile from POST
$userMobile = $_POST['user_mobile'] ?? '';
if (empty($userMobile)) {
    die(json_encode(['error' => 'Mobile number is required']));
}

// Format subscriber ID
$subscriberId = "tel:88" . $userMobile;
$mobileNumber = "88" . $userMobile;

// Save to file (optional debug)
file_put_contents("user_number_fn.txt", $subscriberId);

// Get or create user
$userId = $otpManager->getOrCreateUser($mobileNumber, $subscriberId);

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

// Start logging
$otpManager->logApiStart('otp_request', $url, 'POST');
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
    $otpManager->logApiComplete('', $responseTime, $httpCode, $error);
    echo json_encode(['error' => 'cURL error: ' . $error]);
    curl_close($ch);
    $otpManager->close();
    exit;
}

curl_close($ch);

// Parse response
$response = json_decode($responseJson, true);
$otpManager->logApiComplete($responseJson, $responseTime, $httpCode);

if ($response === null) {
    echo json_encode(['error' => 'Invalid JSON in response', 'raw_response' => $responseJson]);
    $otpManager->close();
    exit;
}

// Save OTP request
$saveResult = $otpManager->saveOTPRequest($userId, $mobileNumber, $subscriberId, $requestData, $response);
$otpRequestId = $saveResult['id'];
$referenceNo = $saveResult['reference_no'];

// Return response
header('Content-Type: application/json');
echo json_encode([
    'database_id' => $otpRequestId,
    'user_id' => $userId,
    'status_code' => $response["statusCode"] ?? null,
    'status_detail' => $response["statusDetail"] ?? null,
    'reference_no' => $referenceNo,
    'version' => $response["version"] ?? null,
    'subscriber_id' => $subscriberId,
    'user_status' => 'pending',
    'expires_in' => '2 minutes'
]);

$otpManager->close();
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

// Database configuration
define('DB_HOST', 'localhost');
define('DB_USER', 'user');
define('DB_PASS', 'pass');
define('DB_NAME', 'name');

// BDApps API configuration
define('APP_ID', 'APP_ID');
define('APP_PASSWORD', 'APP_PASSWORD');

class Database {
    private $conn;
    
    public function __construct() {
        $this->conn = new mysqli(DB_HOST, DB_USER, DB_PASS, DB_NAME);
        if ($this->conn->connect_error) {
            die("Database connection failed: " . $this->conn->connect_error);
        }
        $this->conn->set_charset("utf8mb4");
    }
    
    public function getConnection() {
        return $this->conn;
    }
    
    public function close() {
        $this->conn->close();
    }
}

class OTPVerifier {
    private $conn;
    
    public function __construct($db) {
        $this->conn = $db->getConnection();
    }
    
    public function getPendingOTPRequest($referenceNo) {
        $stmt = $this->conn->prepare("SELECT id, user_id, subscriber_id FROM otp_requests 
            WHERE reference_no = ? 
            AND request_status = 'success' 
            AND verified_at IS NULL 
            AND expires_at > NOW() 
            ORDER BY created_at DESC LIMIT 1");
        $stmt->bind_param("s", $referenceNo);
        $stmt->execute();
        
        $stmt->bind_result($id, $user_id, $subscriber_id);
        
        if ($stmt->fetch()) {
            $data = array(
                'id' => $id,
                'user_id' => $user_id,
                'subscriber_id' => $subscriber_id
            );
            $stmt->close();
            return $data;
        } else {
            $stmt->close();
            return null;
        }
    }
    
    public function saveVerification($otpRequestId, $userId, $otpCode, $referenceNo, $requestPayload, $responsePayload, $isVerified) {
        $statusCode = $responsePayload['statusCode'] ?? null;
        $statusDetail = $responsePayload['statusDetail'] ?? null;
        $requestJson = json_encode($requestPayload);
        $responseJson = json_encode($responsePayload);
        $verifiedAt = $isVerified ? date('Y-m-d H:i:s') : null;
        
        $stmt = $this->conn->prepare("INSERT INTO otp_verifications 
            (otp_request_id, user_id, otp_code, reference_no, request_payload, response_payload, status_code, status_detail, is_verified, verified_at) 
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)");
        
        $stmt->bind_param("iissssssis", 
            $otpRequestId, $userId, $otpCode, $referenceNo, 
            $requestJson, $responseJson, $statusCode, $statusDetail, $isVerified, $verifiedAt
        );
        
        $stmt->execute();
        $verificationId = $stmt->insert_id;
        $stmt->close();
        
        if ($isVerified) {
            $this->conn->query("UPDATE otp_requests SET verified_at = NOW() WHERE id = " . intval($otpRequestId));
            $this->activateUser($userId);
        }
        
        return $verificationId;
    }
    
    private function activateUser($userId) {
        $stmt = $this->conn->prepare("UPDATE users SET status = 'active' WHERE id = ?");
        $stmt->bind_param("i", $userId);
        $stmt->execute();
        $stmt->close();
    }
    
    public function logApiCall($apiType, $endpoint, $requestBody, $responseBody, $httpCode, $error = null) {
        $stmt = $this->conn->prepare("INSERT INTO api_logs 
            (api_type, endpoint, request_method, request_body, response_body, http_code, error_message, created_at) 
            VALUES (?, ?, 'POST', ?, ?, ?, ?, NOW())");
        
        $stmt->bind_param("ssssis", $apiType, $endpoint, $requestBody, $responseBody, $httpCode, $error);
        $stmt->execute();
        $stmt->close();
    }
}

// Initialize
$db = new Database();
$verifier = new OTPVerifier($db);

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
    $db->close();
    exit;
}

// Find pending OTP
$otpRequest = $verifier->getPendingOTPRequest($referenceNo);

if (!$otpRequest) {
    header('Content-Type: application/json');
    echo json_encode([
        'status' => 'error',
        'message' => 'Invalid or expired OTP request. OTP valid for 2 minutes only.'
    ]);
    $db->close();
    exit;
}

$otpRequestId = $otpRequest['id'];
$userId = $otpRequest['user_id'];

// Prepare BDApps API request
$requestData = array(
    "applicationId" => APP_ID,
    "password" => APP_PASSWORD,
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
    $verifier->logApiCall('otp_verify', $url, $requestJson, '', $httpCode, $error);
    
    header('Content-Type: application/json');
    echo json_encode([
        'status' => 'error',
        'message' => 'cURL error: ' . $error
    ]);
    $db->close();
    exit;
}

curl_close($ch);

$response = json_decode($responseJson, true);
$verifier->logApiCall('otp_verify', $url, $requestJson, $responseJson, $httpCode, null);

if ($response === null) {
    header('Content-Type: application/json');
    echo json_encode([
        'status' => 'error',
        'message' => 'Invalid JSON in response',
        'raw_response' => $responseJson
    ]);
    $db->close();
    exit;
}

$isVerified = ($response["statusCode"] === "S1000");
$subscriptionStatus = $response["subscriptionStatus"] ?? null;

$verificationId = $verifier->saveVerification(
    $otpRequestId,
    $userId,
    $otp,
    $referenceNo,
    $requestData,
    $response,
    $isVerified
);

header('Content-Type: application/json');

if ($isVerified) {
    echo json_encode([
        'status' => 'success',
        'message' => 'OTP verified successfully',
        'data' => [
            'verification_id' => $verificationId,
            'reference_no' => $referenceNo,
            'subscriber_id' => $response["subscriberId"] ?? null,
            'subscription_status' => $subscriptionStatus,
            'user_status' => 'active',
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
            'verification_id' => $verificationId,
            'reference_no' => $referenceNo,
            'status_code' => $response["statusCode"] ?? null,
            'status_detail' => $response["statusDetail"] ?? null,
            'version' => $response["version"] ?? null
        ]
    ]);
}

$db->close();
?>
```