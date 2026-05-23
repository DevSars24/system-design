# 📢 HLD Lecture 7 — Notification System Design

A **Notification System** is a vital component in modern applications, responsible for sending critical updates, promotional messages, and security alerts to users across multiple communication channels (Mobile Push, SMS, Email, and In-App).

---

## 📌 1. System Requirements & Core Questions

When designing a notification system, we must clarify the scope by addressing key system requirements.

### Key Questions to Ask in an Interview
1. **What types of notifications do we support?**
   * **Push Notification (Mobile/Web):** High visibility alerts on the device screen (iOS/Android/Web).
   * **SMS Notification:** SMS messages sent to the user's phone number.
   * **Email Notification:** Rich HTML or plain-text emails.
   * **In-App Notification:** Messages shown while the user is actively using the app (e.g., notification center).
2. **What is the system's real-time requirement?**
   * **Real-time System:** Notifications must be delivered *immediately* (e.g., OTP for 2FA).
   * **Soft Real-time System:** We try to deliver notifications as soon as possible, but brief delays are acceptable under high load, network partitions, or 3rd party downtimes.
   * *Decision:* A notification system is designed as a **Soft Real-time System** because it depends on 3rd party external networks.
3. **What devices/platforms do we target?**
   * iOS, Android, and Web browsers (macOS, Windows, Linux).

### Functional Requirements
* **Send Notifications:** Support sending notifications via Push, SMS, and Email.
* **Notification Preferences:** Users can opt-in/opt-out of specific notification types.
* **Notification History:** Users can view their past notification history.
* **Templates:** support dynamic content using predefined templates.

### Non-Functional Requirements
* **High Availability:** System must be available to accept notification requests 24/7.
* **Low Latency:** Delivery should happen within seconds for soft real-time.
* **Reliability (No Data Loss):** Important notifications (like OTPs) must never be lost.
* **Scalability:** Must handle sudden spikes (e.g., flash sales, breaking news alerts).
* **Loose Coupling:** The core application services should not block when triggering a notification.

---

## 📐 2. Core Entities & Basic Flow

A notification system comprises several core entities working together:

```
┌──────────────┐      ┌──────────────────────┐      ┌────────────┐      ┌──────────┐
│   Service    │ ───> │ Notification Service │ ───> │  Executor  │ ───> │  Client  │
│  (Trigger)   │      │ (Payloads/Templates) │      │ (Provider) │      │ (Device) │
└──────────────┘      └──────────────────────┘      └────────────┘      └──────────┘
```

1. **Service (Trigger):** Any microservice (e.g., Order Service, Auth Service) or scheduled Cron-job that triggers a notification request.
2. **Notification Service:** The entry point that processes the request, resolves user preferences, generates templates, and validates input.
3. **Executor:** The subsystem/workers that communicate with 3rd-party gateways to dispatch notifications.
4. **Client:** The end-user's device receiving the notification.

---

## 📊 3. Back-of-the-Envelope Calculations (BEC)

Before designing the architecture, we estimate the system scale to size our database, queues, and bandwidth.

### Assumptions:
* **Daily Active Users (DAU):** $1\text{ Million}$
* **Average Notifications per User per Day:** $10$
* **Total Notifications/Day:** $1\text{ Million} \times 10 = 10\text{ Million notifications/day}$

---

### A. Storage Calculations

Every notification metadata log must be stored in the database for history and analytics.
* **Average size of 1 notification record:** $200\text{ bytes}$ (includes `userId`, `notificationId`, `channel`, `content_snippet`, `status`, `timestamp`).

$$\text{Daily Storage} = 10\text{ Million} \times 200\text{ bytes}$$
$$\text{Daily Storage} = 2 \times 10^9\text{ bytes} \approx 2\text{ GB/day}$$

$$\text{Monthly Storage} = 2\text{ GB/day} \times 30\text{ days} = 60\text{ GB/month}$$
$$\text{Yearly Storage} = 60\text{ GB/month} \times 12\text{ months} = 720\text{ GB/year}$$

---

### B. Bandwidth & Load Calculations

Assume the raw payload size (metadata + body + headers) sent to our server averages $1\text{ KB}$ per notification.

$$\text{Total Daily Data Transmitted} = 10\text{ Million} \times 1\text{ KB} = 10\text{ GB/day}$$

#### Average Queries Per Second (QPS):
$$\text{Seconds in a day} = 24 \times 3600 = 86,400\text{ seconds}$$
$$\text{Average QPS} = \frac{10,000,000\text{ notifications}}{86,400\text{ seconds}} \approx 115.7\text{ req/sec}$$

#### Peak QPS (Assuming $3\text{x}$ average load):
$$\text{Peak QPS} = 115.7 \times 3 \approx 350\text{ req/sec}$$

#### Bandwidth Usage:
$$\text{Average Bandwidth} = \frac{10\text{ Million} \times 1\text{ KB}}{86,400\text{ seconds}} \approx 115.7\text{ KB/second}$$

---

## 🛠️ 4. API Interface Design & DB Schema

### A. API Endpoints

#### 1. Retrieve User Settings & Info
Retrieves user contact information and specific opt-in/opt-out notification channel permissions.

* **HTTP Method:** `GET`
* **Path:** `/api/v1/users/{userId}`
* **Response (200 OK):**
```json
{
  "userId": "usr_101",
  "userName": "Aditya",
  "email": "aditya@example.com",
  "mobile": "+919876543210",
  "notificationPermissions": {
    "PUSH": true,
    "EMAIL": true,
    "SMS": false
  }
}
```

#### 2. Send Notification
Endpoint triggered by internal microservices to dispatch a notification.

* **HTTP Method:** `POST`
* **Path:** `/api/v1/notifications/send/{userId}`
* **Request Body:**
```json
{
  "from": "system@app.com",
  "to": "+919876543210",
  "subject": "Order Shipped!",
  "content": "Hi Aditya, your order #10928 has been dispatched.",
  "type": "SMS" 
}
```
* **Response (200 OK):**
```json
{
  "status": "queued",
  "notificationId": "notif_301"
}
```

#### 3. Fetch Notification History
Fetches log of notifications sent to a specific user with filtering capability.

* **HTTP Method:** `GET`
* **Path:** `/api/v1/notifications/history`
* **Query Parameters:**
  * `userId`: `usr_101`
  * `type`: `SMS` (Optional)
  * `fromDate`: `2026-05-20` (Optional)
* **Response (200 OK):**
```json
[
  {
    "notificationId": "notif_301",
    "status": "delivered",
    "type": "SMS",
    "content": "Hi Aditya, your order #10928 has been dispatched.",
    "sentAt": "2026-05-23T12:00:00Z"
  },
  {
    "notificationId": "notif_289",
    "status": "failed",
    "type": "EMAIL",
    "content": "Weekly Newsletter",
    "sentAt": "2026-05-21T09:15:00Z"
  }
]
```

---

### B. Database Schema Design (SQL DDL)

To maintain consistent user records, dynamic templates, and detailed logs, we use a relational database structure.

```sql
-- 1. Users Table
CREATE TABLE users (
    user_id VARCHAR(50) PRIMARY KEY,
    username VARCHAR(100) NOT NULL,
    email VARCHAR(150) UNIQUE,
    mobile_number VARCHAR(20) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 2. User Notification Permissions (Preferences)
CREATE TABLE user_permissions (
    user_id VARCHAR(50) PRIMARY KEY,
    allow_push BOOLEAN DEFAULT TRUE,
    allow_email BOOLEAN DEFAULT TRUE,
    allow_sms BOOLEAN DEFAULT TRUE,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);

-- 3. Notification Templates
CREATE TABLE notification_templates (
    template_id VARCHAR(50) PRIMARY KEY,
    template_name VARCHAR(100) NOT NULL,
    channel_type VARCHAR(20) NOT NULL, -- 'PUSH', 'SMS', 'EMAIL'
    subject_template VARCHAR(255),    -- Used for emails
    body_template TEXT NOT NULL,       -- e.g., "Hello {{userName}}, your code is {{code}}"
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 4. Notification History (Log table for Audits and Analytics)
CREATE TABLE notification_history (
    notification_id VARCHAR(50) PRIMARY KEY,
    user_id VARCHAR(50) NOT NULL,
    channel_type VARCHAR(20) NOT NULL,
    recipient_destination VARCHAR(255) NOT NULL, -- email address, phone number, or push token
    status VARCHAR(20) NOT NULL,                  -- 'QUEUED', 'SENT', 'FAILED', 'DELIVERED'
    retry_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Add Index for history lookups (optimizes GET /notifications/history)
CREATE INDEX idx_user_history ON notification_history(user_id, created_at DESC);
```

---

## 🚫 5. Problems with the Initial (Synchronous) Design

In a simple monolithic design, we would have services directly calling the Notification Server, which then synchronously calls external 3rd-party services to dispatch notifications.

```
┌───────────┐      ┌──────────────┐      ┌───────────┐      ┌──────────┐
│ Service 1 │ ───> │ Notification │ ───> │ 3rd-Party │ ───> │  Client  │
└───────────┘      │    Server    │      │ API (SMS) │      └──────────┘
                   └──────────────┘      └───────────┘
                         ▲                     │ (Blocks execution)
                         └─────── Wait ────────┘
```

### Why this design fails in production:
1. **Single Point of Failure (SPOF):** If the notification server crashes, the entire system's ability to alert users fails.
2. **Synchronous Bottleneck & Latency:** 3rd party APIs (like Twilio, Mailchimp, Apple APNs) can take hundreds of milliseconds or even seconds to respond. The internal application service is blocked waiting for the API call to complete, degrading response times.
3. **Hard to Scale:** During peak traffic events, if the notification server is saturated with requests, it cannot handle them asynchronously, leading to memory exhausts or thread pools running dry.
4. **No Retry Mechanism:** If a 3rd party gateway is down or rate-limiting us, the notification fails permanently. There is no automated buffer to retry.
5. **Tight Coupling:** The calling services must wait for the notification to be processed, meaning an email failure could block a checkout transaction.

---

## 🚀 6. Scaled Asynchronous Architecture

To build a production-grade system, we decouple the notification pipeline using **Message Queues (MQ)**, **Workers**, **Caching**, and **Databases**.

### Architectural Diagram

```
                              ┌─────────────────────────────────────────┐
                              │           Redis Cache (Fast)            │
                              │ (User Info, Permissions, Templates)     │
                              └────────────────────┬────────────────────┘
                                                   │ Check Cache
                                                   ▼
┌──────────────┐              ┌─────────────────────────────────────────┐
│  Services 1  │ ──Trigger──> │            Publisher Server             │
├──────────────┤              │   - Validates user & check permissions  │
│  Services N  │ ──Trigger──> │   - Compiles template placeholders      │
└──────────────┘              └────────────────────┬────────────────────┘
                                                   │ Push to channel MQs
                                                   ▼
                                ┌───────────────────────────────────────┐
                                │          Message Queues (MQs)         │
                                │  ┌──────────────┐   ┌──────────────┐  │
                                │  │  APNs Queue  │   │  FCM Queue   │  │
                                │  └──────┬───────┘   └──────┬───────┘  │
                                │  ┌──────┴───────┐   ┌──────┴───────┐  │
                                │  │ Twilio Queue │   │Mailchimp Q.  │  │
                                │  └──────┬───────┘   └──────┬───────┘  │
                                └─────────┼──────────────────┼──────────┘
                                          │ Consume          │ Consume
                                          ▼                  ▼
                                ┌──────────────────┐┌──────────────────┐
                                │  Twilio Workers  ││Mailchimp Workers │
                                └─────────┬────────┘└────────┬─────────┘
                                          │                  │
                         ┌─────────Retry  │                  │  Retry─────────┐
                         ▼                ▼                  ▼                ▼
                    ┌─────────┐     ┌───────────┐      ┌───────────┐     ┌─────────┐
                    │ Twilio  │     │ 3rd-Party │      │ 3rd-Party │     │Mailchimp│
                    │   DLQ   │     │ API (SMS) │      │API (Email)│     │   DLQ   │
                    └─────────┘     └─────┬─────┘      └─────┬─────┘     └─────────┘
                                          │                  │
                                          ▼                  ▼
                                ┌───────────────────────────────────────┐
                                │      Notification Log DB / Cache      │
                                └──────────────────┬────────────────────┘
                                                   │ Writes Logs
                                                   ▼
                                ┌───────────────────────────────────────┐
                                │           Analytics Service           │
                                └───────────────────────────────────────┘
```

---

### Step-by-Step Flow:
1. **Trigger:** A service sends a request to the Publisher Server.
2. **Caching Check:** The Publisher checks **Redis Cache** first to retrieve user contact details, opt-in preferences, and templates. If not in cache, it retrieves them from the Database and updates Redis.
3. **Template Compilation:** The Publisher merges the request data with the template (e.g. replacing `{{userName}}` with `"Aditya"`).
4. **Enqueueing:** Publisher publishes a message containing all details to a specific **Message Queue** depending on the channel type (e.g. Twilio Queue for SMS, Mailchimp Queue for Email).
5. **Consumption:** Dedicated Workers (e.g. Twilio Workers) pull messages from their respective queues.
6. **Execution:** Workers make asynchronous HTTP calls to the 3rd party API.
7. **Retries:** If the 3rd party gateway fails, the message is placed back into the queue for a retry. If it repeatedly fails, it goes to the **Dead Letter Queue (DLQ)**.
8. **Logging & Analytics:** On success or final failure, workers log the status to the **Notification Log DB**, which feeds the **Analytics Service** dashboard.

---

## 🛡️ 7. Resilience, Fault Tolerance, & Advanced Concepts

### A. Robust Retry Mechanism
Workers calling 3rd-party APIs will inevitably hit network timeouts or rate-limiting responses (`HTTP 429`). To handle this:
* **Exponential Backoff:** If an API call fails, the worker retries after an exponentially increasing delay (e.g., $2\text{s}$, $4\text{s}$, $8\text{s}$, $16\text{s}$).
* **Jitter:** Add a random delay factor (noise) to prevent all retrying workers from hitting the 3rd-party API simultaneously (thundering herd problem).
* **Dead Letter Queue (DLQ):** After a maximum number of retries (e.g., 5 times), the message is routed to a DLQ. DevOps teams are alerted to manually investigate (e.g., wrong phone number format or invalid API credentials).

#### Example Worker Retrying Pseudocode (Python):
```python
import time
import random

MAX_RETRIES = 5
BASE_DELAY = 2  # seconds

def send_with_retry(notification_message):
    retries = 0
    while retries < MAX_RETRIES:
        try:
            # Call third party service (e.g., Twilio API)
            response = call_third_party_gateway(notification_message)
            if response.status_code == 200:
                update_db_status(notification_message.id, "DELIVERED")
                return True
            elif response.status_code == 429: # Rate limited
                print("Rate limited by provider. Retrying...")
            else:
                print("Provider returned error. Retrying...")
        except Exception as e:
            print(f"Network error: {e}. Retrying...")
        
        retries += 1
        # Exponential Backoff with Jitter
        delay = (BASE_DELAY ** retries) + random.uniform(0.5, 1.5)
        print(f"Waiting {delay:.2f} seconds before retry...")
        time.sleep(delay)
        
    # Exceeded max retries -> Move to DLQ
    move_to_dead_letter_queue(notification_message)
    update_db_status(notification_message.id, "FAILED")
```

---

### B. Notification Templates
Storing entire message text for every request wastes bandwidth and database space. Instead, use templates stored in the database/cache:
* **Template Definition:** `Hello {{userName}}, your OTP is {{otp}}. Do not share it.`
* **Request Payload:**
```json
{
  "userId": "usr_101",
  "templateId": "tmpl_otp_verification",
  "placeholders": {
    "userName": "Aditya",
    "otp": "482091"
  }
}
```
* **Benefit:** Smaller network footprint, and product managers can update copy changes in the database without modifying backend code.

---

### C. Rate Limiting & User-Level Spam Prevention
To prevent spamming users (e.g., a buggy system loop sending 50 emails to a customer in 2 minutes):
* **Rate Limiter on API Gateway:** Limits requests coming from internal services.
* **Frequency Cap on User Level:** Keep a counter in Redis for each user ID: `notif:limit:usr_101`.
* If a new notification is requested, check if the user has already received more than the allowed threshold (e.g., max 3 notifications per minute). If exceeded, drop or delay the notification.

---

### D. Security & Authentication
* **Internal Security:** Use API Key validation or JWTs to authenticate microservices communicating with the notification server.
* **3rd Party Credentials:** Store provider API keys (Twilio auth token, Sendgrid key) in secure secret managers (e.g., AWS Secrets Manager, HashiCorp Vault) rather than hardcoding.

---

### E. Monitoring & Observability
Key metrics to track in production:
* **Queue Size / Depth:** If the FCM queue is growing rapidly, it indicates workers are falling behind or Firebase is experiencing downtime.
* **Delivery Latency:** The time elapsed from publisher ingest to client receipt.
* **Error Rate:** Percentage of failed API calls grouped by 3rd-party provider.
* **System Saturation:** CPU and memory utilization of worker instances to trigger auto-scaling.
