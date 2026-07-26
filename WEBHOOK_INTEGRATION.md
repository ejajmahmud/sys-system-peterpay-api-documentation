# PeterPay API: Webhook Integration and Best Practices

Webhooks are an essential part of integrating with PeterPay, allowing your application to receive real-time notifications about payment events. This guide will walk you through setting up and securely handling PeterPay webhooks.

## 1. What are Webhooks?

**Webhooks are automated messages sent from applications when an event occurs.** In the context of PeterPay, webhooks are used to notify your server about the status of a payment (e.g., `payment.completed`, `payment.failed`). This is crucial for maintaining the state of transactions in your system without constantly polling the PeterPay API for updates.

**Why use Webhooks?**

*   **Real-time Updates:** Receive instant notifications when a payment status changes.
*   **Reduced API Calls:** Avoid unnecessary API calls to check payment status, saving resources and reducing latency.
*   **Reliability:** Webhooks are designed to be resilient, often retrying failed deliveries to ensure your system eventually receives the notification.

## 2. Configuring Your Webhook URL

To receive webhook notifications, you need to provide PeterPay with a publicly accessible URL on your server. This URL will be the endpoint where PeterPay sends POST requests containing event data.

1.  **Log in to your PeterPay Dashboard.**
2.  Navigate to the **Settings** or **API Configuration** section.
3.  Locate the **Webhook URL** field.
4.  Enter the full URL of your webhook endpoint (e.g., `https://yourdomain.com/api/peterpay-webhook`).
5.  **Save** your changes.

**Important Considerations:**

*   **HTTPS is Mandatory:** Your webhook URL **must** use HTTPS to ensure the security and integrity of the data sent from PeterPay to your server.
*   **Publicly Accessible:** The URL must be accessible from the public internet. Localhost or URLs behind firewalls will not work unless properly exposed.
*   **Multiple Endpoints:** PeterPay supports multiple webhook URLs. If you have different applications or environments that need to receive notifications, you can add them all in your dashboard.

## 3. Handling Webhook Notifications

When PeterPay sends a webhook, it will be a `POST` request to your configured URL with a JSON payload in the request body.

### Example Webhook Payload

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

Your webhook endpoint should:

1.  **Receive the POST request.**
2.  **Parse the JSON payload.**
3.  **Verify the webhook signature** (see Section 4).
4.  **Process the event:** Update your database, trigger business logic, send confirmations, etc.
5.  **Respond with a `200 OK` status code:** This tells PeterPay that you successfully received and processed the webhook. Any other status code (e.g., `4xx`, `5xx`) will indicate a failure, and PeterPay will retry sending the webhook.

## 4. Webhook Security: Signature Verification

**Signature verification is the most critical step in handling webhooks.** It ensures that the incoming request genuinely originated from PeterPay and that the payload has not been tampered with during transit. PeterPay includes an `X-PeterPay-Signature` header in every webhook request.

This signature is an HMAC SHA256 hash of the raw request body, signed with your unique **API Secret** (which you can find in your PeterPay Dashboard).

### How to Verify the Signature

To verify the signature, you need to:

1.  Retrieve the raw request body.
2.  Retrieve the `X-PeterPay-Signature` header value.
3.  Retrieve your **API Secret** from your secure environment variables.
4.  Compute the HMAC SHA256 hash of the raw request body using your API Secret.
5.  Compare your computed hash with the `X-PeterPay-Signature` header.

#### PHP Example for Signature Verification

```php
<?php
// Get the raw request body
$payload = file_get_contents("php://input");

// Get the signature from the header
$signature = $_SERVER["HTTP_X_PETERPAY_SIGNATURE"];

// Your PeterPay API Secret (store securely, e.g., in environment variables)
$secret = getenv("PETERPAY_API_SECRET"); // Or from a config file

// Compute the HMAC SHA256 hash
$calculated_signature = hash_hmac("sha256", $payload, $secret);

// Compare the calculated signature with the received signature
if (hash_equals($calculated_signature, $signature)) {
    // Signature is valid, process the webhook event
    $event_data = json_decode($payload, true);
    
    // Example: Log the event or update your database
    error_log("Webhook received: " . $event_data["event"] . " for order " . $event_data["order_id"]);
    
    // Send a 200 OK response
    http_response_code(200);
    echo "OK";
} else {
    // Signature is invalid, reject the request
    error_log("Invalid webhook signature!");
    http_response_code(403); // Forbidden
    echo "Forbidden";
}
?>
```

#### Node.js / JavaScript Example for Signature Verification

```javascript
const crypto = require("crypto");

export default function handler(req, res) {
  if (req.method !== "POST") {
    return res.status(405).send("Method Not Allowed");
  }

  const signature = req.headers["x-peterpay-signature"];
  const secret = process.env.PETERPAY_API_SECRET; // Store securely in environment variables

  if (!signature || !secret) {
    console.error("Missing webhook signature or secret.");
    return res.status(400).send("Bad Request");
  }

  // Get the raw request body (important: do not parse JSON before verification)
  const payload = JSON.stringify(req.body); // Assuming body-parser has already parsed it, stringify back to raw form
  // For raw body, you might need to configure your server to get the raw body stream
  // In Next.js API routes, you might need `bodyParser: false` and read stream manually.

  const hmac = crypto.createHmac("sha256", secret);
  hmac.update(payload);
  const calculatedSignature = hmac.digest("hex");

  if (calculatedSignature === signature) {
    // Signature is valid, process the webhook event
    const eventData = req.body;
    console.log(`Webhook received: ${eventData.event} for order ${eventData.order_id}`);
    
    // Respond with 200 OK
    res.status(200).send("OK");
  } else {
    // Signature is invalid
    console.error("Invalid webhook signature!");
    res.status(403).send("Forbidden");
  }
}
```

**Note on `req.body` in Node.js:** When using frameworks like Express or Next.js, `req.body` is often automatically parsed. For signature verification, you need the *raw* request body before any parsing. You might need to adjust your server configuration to access the raw body or stringify the parsed body back to its original form if it's guaranteed to be the exact original payload.

## 5. Best Practices for Webhook Endpoints

*   **Respond Quickly:** Your webhook endpoint should respond with a `200 OK` as quickly as possible. If your processing logic is complex or time-consuming, queue the event for asynchronous processing and respond immediately.
*   **Idempotency:** Design your webhook handler to be idempotent. This means that processing the same webhook event multiple times should have the same result as processing it once. PeterPay may retry sending webhooks, so your system should be able to handle duplicates gracefully.
*   **Error Logging and Monitoring:** Implement robust logging for all incoming webhooks and any errors that occur during processing. Set up monitoring and alerts to be notified of failures.
*   **Security:** Beyond signature verification, ensure your webhook endpoint is protected against other common web vulnerabilities (e.g., SQL injection, XSS).
