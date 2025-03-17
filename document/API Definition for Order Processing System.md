```yaml
layout: post
title: "API design"
date: 2025-03-16 16:40:59
abstract: This design follows the event-driven architecture, making the system more loosely coupled, scalable, and fault-tolerant.
tags: [API, event-driven]
```

# API Definition for Order Processing System

____

## 1. Order Service

### 1.1 Create Order Endpoint

- **Method:** POST

- **URL:** `/api/orders/create`

- **Request Example:**
  
  ```
  {
      "userId": "123456",
      "token": "user authentication token",
      "signature": "digital signature",
      "orderAmount": 100.50,
      "items": [
          {
              "productId": "P001",
              "quantity": 2
          },
          {
              "productId": "P002",
              "quantity": 1
          }
      ]
  }
  ```

- **Response Example (Success):**
  
  ```
  {
   "code": 200,
   "message": "Order created successfully",
   "orderId": "ORD123456",
   "status": "pending" // Indicates awaiting inventory confirmation
  }
  ```

- **Notes:**
  
  - The backend should validate the data:
    - **Order Amount:** Must not be negative.
    - **Product ID:** Must be valid.
    - **Quantity:** Must be a positive integer.
  - Token-based authentication, digital signature, and CSRF prevention are used.
  - Upon successful creation, an order event is pushed to the message queue for the inventory service.

<br />

## 2. Inventory Service

### 2.1 Lock Inventory Endpoint

- **Method:** POST

- **URL:** `/api/inventory/lock`

- **Request Example:**
  
  ```
  {
      "orderId": "ORD123456",
      "items": [
          {
              "productId": "P001",
              "quantity": 2
          },
          {
              "productId": "P002",
              "quantity": 1
          }
      ]
  }
  ```
+ **Response Example (Success):**
  
  ```
  {
      "code": 200,
      "message": "Inventory locked successfully",
      "locked": true
  }
  ```

+ **Response Example (Failure):**
  
  ```
  {
      "code": 400,
      "message": "Insufficient inventory",
      "locked": false
  }
  ```
- **Notes:**
  
  - This endpoint is generally **triggered by the message queue when an order creation event is received**.
  
  - Ensure **idempotency** to prevent duplicate inventory deductions.

<br />

## 3. Payment Service

### 3.1 Process Payment Endpoint

- **Method:** POST

- **URL:** `/api/payment/process`

- **Request Example:**
  
  ```
  {
      "orderId": "ORD123456",
      "userId": "123456",
      "paymentAmount": 100.50,
      "paymentMethod": "credit_card",  // e.g., credit_card, paypal, etc.
      "paymentDetails": {
          "cardNumber": "4111111111111111",
          "expiryDate": "12/25",
          "cvv": "123"
      }
  }
  ```
+ **Response Example (Success):**
  
  ```
  {
      "code": 200,
      "message": "Payment successful",
      "paymentStatus": "completed",
      "transactionId": "TXN789012"
  }
  ```

+ **Response Example (Failure):**
  
  ```
  {
      "code": 400,
      "message": "Payment failed due to insufficient funds",
      "paymentStatus": "failed"
  }
  ```
- **Notes:**
  
  - This endpoint is triggered after the inventory service confirms the inventory lock.
  
  - On successful payment, a message is sent to notify other services (such as the notification service) about the updated order status.
  
  - The endpoint simulates an asynchronous payment process, including handling callbacks from a payment gateway.

<br />

## 4. Notification Service

### 4.1 Send Notification Endpoint

- **Method:** POST

- **URL:** `/api/notification/send`

- **Request Example:**
  
  ```
  {
      "userId": "123456",
      "notificationType": "email",  // Options: "email" or "sms"
      "subject": "Order Status Update",
      "message": "Your order ORD123456 has been successfully paid."
  }
  ```

- **Request Example:**
  
  ```
  {
      "code": 200,
      "message": "Notification sent successfully"
  }
  ```

- **Response Example:**
  
  ```
  {
      "code": 200,
      "message": "Notification sent successfully"
  }
  ```

- **Notes:**
  
  - This endpoint is triggered by the message queue when payment results or order status updates are received.
  - It handles sending notifications via email or SMS and can also manage delayed tasks (e.g., order timeout alerts).
