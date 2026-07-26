# PeterPay API Documentation 🚀

[![GitHub stars](https://img.shields.io/github/stars/King-pe/PeterPay-API-Documentation?style=for-the-badge&color=yellow)](https://github.com/King-pe/PeterPay-API-Documentation/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/King-pe/PeterPay-API-Documentation?style=for-the-badge&color=blue)](https://github.com/King-pe/PeterPay-API-Documentation/network/members)
[![Donate](https://img.shields.io/badge/Donate-Orange?style=for-the-badge&logo=paypal&logoColor=white)](https://www.peterpay.link/pay?code=844ac71e)

Welcome to the official PeterPay API documentation. PeterPay allows you to seamlessly integrate mobile money payments (M-Pesa, Tigo Pesa, Airtel Money, Halotel, etc.) into your website, mobile app, or backend system.

---

## 📌 Introduction
PeterPay provides a simple REST API to initiate USSD Push (STK Push) payments and verify transaction statuses in real-time.

- **Base URL:** `https://www.peterpay.link/api/v1`
- **Dashboard:** [Get your API Key here](https://www.peterpay.link/dashboard.php)

---

## 🔐 Authentication
All API requests must include the `X-API-KEY` header to identify your account.

**Header Example:**
```http
X-API-KEY: pk_live_xxxxxxxxxxxxxxxxxxxxxxxx
```

---

## 💳 1. Create Order (Initiate Payment)
Use this endpoint to initiate a payment request. The customer will receive a USSD Push (STK Push) notification on their phone to enter their PIN.

- **Method:** `POST`
- **Endpoint:** `/create_order`

### Request Parameters
| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `amount` | Float | Yes | Payment amount (Minimum: 500 TZS). |
| `buyer_phone` | String | Yes | Customer\'s phone number in international format (e.g., 2557XXXXXXXX). |
| `buyer_name` | String | No | Customer\'s name. |
| `buyer_email` | String | No | Customer\'s email address. |

### Implementation Examples

#### PHP (Using cURL)
```php
function peterpay_request($endpoint, $payload){

    $url = PETERPAY_API_URL . $endpoint;

    $jsonData = json_encode($payload);

    $ch = curl_init($url);

    curl_setopt_array($ch,[

        CURLOPT_URL => $url,

        CURLOPT_RETURNTRANSFER => true,

        CURLOPT_POST => true,

        CURLOPT_CUSTOMREQUEST => "POST",

        CURLOPT_POSTFIELDS => $jsonData,

        CURLOPT_HTTPHEADER => [

            'Content-Type: application/json',

            'Content-Length: ' . strlen($jsonData),

            'Accept: application/json',

            'X-API-KEY: ' . PETERPAY_API_KEY
        ],

        CURLOPT_SSL_VERIFYPEER => false,

        CURLOPT_TIMEOUT => 30,

        CURLOPT_CONNECTTIMEOUT => 10
    ]);

    $response = curl_exec($ch);

    $http_code = curl_getinfo($ch, CURLINFO_HTTP_CODE);

    $curl_error = curl_error($ch);

    curl_close($ch);

    if($response === false){

        return [

            'status' => 'error',

            'message' => $curl_error
        ];
    }

    $decoded = json_decode($response,true);

    if(!$decoded){

        return [

            'status' => 'error',

            'message' => 'Invalid JSON',

            'raw' => $response
        ];
    }

    return $decoded;
}
```

#### JavaScript (Node.js / Frontend)
```javascript
const axios = require(\'axios\');

const data = {
    amount: 1000,
    buyer_phone: \'255753033342\',
    buyer_name: \'John Doe\'
};

axios.post(\'https://www.peterpay.link/api/v1/create_order\', data, {
    headers: {
        \'Content-Type\': \'application/json\',
        \'X-API-KEY\': \'YOUR_API_KEY\'
    }
})
.then(response => console.log(response.data))
.catch(error => console.error(error));
```

### Success Response
```json
{
  "status": "success",
  "message": "Order created",
  "data": {
    "order_id": "API-ABC123XYZ",
    "payment_status": "PENDING"
  }
}
```

---

## 🔍 2. Check Status (Verification)
Verify if a transaction was successful, failed, or is still pending.

- **Method:** `POST`
- **Endpoint:** `/order_status`

**Request Body:**
```json
{
  "order_id": "API-ABC123XYZ"
}
```

**Response Example:**
```json
{
  "status": "success",
  "data": {
    "order_id": "API-ABC123XYZ",
    "payment_status": "COMPLETED",
    "amount": 1000,
    "currency": "TZS",
    "reference": "SP123456789"
  }
}
```

---

## 🪝 3. Webhooks (Callbacks)
PeterPay sends real-time notifications to your server when a payment is completed. Configure your Webhook URL in the PeterPay Dashboard.

### Payload Example
```json
{
  "event": "payment.completed",
  "order_id": "API-ABC123XYZ",
  "amount": 1000,
  "currency": "TZS",
  "status": "COMPLETED",
  "customer_phone": "255712345678",
  "timestamp": "2024-02-14T10:30:00+03:00"
}
```

### Security (Signature Verification)
To ensure requests are from PeterPay, verify the `X-PeterPay-Signature` header. It is an HMAC SHA256 hash of the payload using your **API Secret**.

#### PHP Verification
```php
$payload = file_get_contents(\'php://input\');
$signature = $_SERVER[\'HTTP_X_PETERPAY_SIGNATURE\'];
$secret = \'YOUR_API_SECRET\';

$calculated = hash_hmac(\'sha256\', $payload, $secret);

if (hash_equals($calculated, $signature)) {
    // Request is valid
}
```

---

## ⚡ Vercel Integration (Next.js / Node.js)
If you are using Vercel, you can easily handle payments using Serverless Functions (API Routes).

### Example: `/api/pay.js`
```javascript
export default async function handler(req, res) {
  if (req.method !== \'POST\') return res.status(405).end();

  const response = await fetch(\'https://www.peterpay.link/api/v1/create_order\', {
    method: \'POST\',
    headers: {
      \'Content-Type\': \'application/json\',
      \'X-API-KEY\': process.env.PETERPAY_API_KEY
    },
    body: JSON.stringify(req.body)
  });

  const data = await response.json();
  res.status(200).json(data);
}
```
*Note: Ensure you add `PETERPAY_API_KEY` to your Vercel Environment Variables.*

---

## 📚 Additional Documentation

For more in-depth guides on securing your integration and handling webhooks, please refer to these dedicated documents:

*   [**Security and SSL Configuration Guide**](SECURITY_AND_SSL.md): Learn about securing your API keys, using HTTPS, and obtaining free SSL certificates.
*   [**Webhook Integration and Best Practices**](WEBHOOK_INTEGRATION.md): A comprehensive guide on setting up, handling, and securing PeterPay webhooks.

---

## 📦 SDKs & Plugins
- **PHP SDK:** Included in the `PeterPaySDK.php` file.
- **WooCommerce:** [Download Plugin](https://www.peterpay.link/peterpay-woocommerce.zip)
- **WHMCS:** [Download Gateway](https://www.peterpay.link/peterpay-whmcs.zip)

---

## 🤝 Support & Contributions
If you find this documentation helpful or wish to support the development of PeterPay, consider donating:

[![Donate](https://img.shields.io/badge/Donate-Orange?style=for-the-badge&logo=paypal&logoColor=white)](https://www.peterpay.link/pay?code=844ac71e)

### Usage Policy
> **Note:** Use of this code is subject to a contribution of **15,000 TZS** via the donate button. Unauthorized modification of the core logic is strictly forbidden. Please notify the developer upon implementation.

---

## 👥 Contributors
Thanks to everyone contributing to this project!

<a href="https://github.com/King-pe/PeterPay-API-Documentation/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=King-pe/PeterPay-API-Documentation" />
</a>

---

**Developed by [Peter Joram](https://www.peterpay.link)**
