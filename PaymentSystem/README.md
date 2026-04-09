# 💳 Payment System - Low Level Design (LLD)

------------------------------------------------------------------------

## 📌 Overview

Production-grade payment system focusing on **correctness, idempotency,
retries, and extensibility**.

------------------------------------------------------------------------

# 🎯 Requirements

## ✅ Functional

-   Create payment
-   Support Card / UPI / Wallet
-   Track lifecycle
-   Retry failures
-   Prevent duplicate charges
-   Maintain transaction history
-   Update ledger

## ⚙️ Non-Functional

-   Strong Consistency
-   Reliability
-   Scalability
-   Low Latency
-   Auditability

------------------------------------------------------------------------

# 🧱 Core Entities

## Payment

-   PaymentId, UserId, Amount, Currency
-   Status (INIT, PROCESSING, SUCCESS, FAILED)
-   RetryCount, PayeeId

## PaymentTransaction (Append-only)

-   TransactionId, PaymentId
-   Status, GatewayTxnId, Response

## IdempotencyRecord

-   IdempotencyKey → PaymentId, Response

## LedgerEntry

-   EntryId, PaymentId
-   FromAccount, ToAccount, Amount

------------------------------------------------------------------------

# 🧩 Interfaces

``` csharp
public interface IPaymentProcessor
{
    PaymentResult Process(Payment payment);
}

public interface IGatewayClient
{
    GatewayResponse Charge(GatewayRequest request);
    GatewayStatus GetStatus(string gatewayTxnId);
}

public interface IIdempotencyService
{
    PaymentResponse Get(string key);
    void Save(string key, PaymentResponse response);
}

public interface IRetryService
{
    void ScheduleRetry(Payment payment, TimeSpan delay);
}

public interface ILedgerService
{
    void Apply(Payment payment);
}
```

------------------------------------------------------------------------

# 🧠 Classes & Responsibilities

## 1. PaymentService (Orchestrator)

### Responsibilities:

-   Entry point
-   Idempotency validation
-   Payment lifecycle management
-   Retry handling

``` csharp
public class PaymentService
{
    public PaymentResponse CreatePayment(PaymentRequest request);
    private void ProcessPayment(Payment payment);
    private void HandleFailure(Payment payment);
    private void RetryPayment(Payment payment);
}
```

------------------------------------------------------------------------

## 2. PaymentProcessor (Strategy)

### Responsibility:

-   Payment method-specific logic

``` csharp
public class CardProcessor : IPaymentProcessor
{
    public PaymentResult Process(Payment payment);
}

public class UpiProcessor : IPaymentProcessor
{
    public PaymentResult Process(Payment payment);
}
```

------------------------------------------------------------------------

## 3. GatewayClient

### Responsibility:

-   External API interaction

``` csharp
public class GatewayClient : IGatewayClient
{
    public GatewayResponse Charge(GatewayRequest request);
    public GatewayStatus GetStatus(string gatewayTxnId);
}
```

------------------------------------------------------------------------

## 4. IdempotencyService

### Responsibility:

-   Prevent duplicate requests

``` csharp
public class IdempotencyService : IIdempotencyService
{
    public PaymentResponse Get(string key);
    public void Save(string key, PaymentResponse response);
}
```

------------------------------------------------------------------------

## 5. RetryService

### Responsibility:

-   Schedule retries using scheduler

``` csharp
public class RetryService : IRetryService
{
    public void ScheduleRetry(Payment payment, TimeSpan delay);
}
```

------------------------------------------------------------------------

## 6. LedgerService

### Responsibility:

-   Handle debit/credit (idempotent)

``` csharp
public class LedgerService : ILedgerService
{
    public void Apply(Payment payment);
}
```

------------------------------------------------------------------------

## 7. Repository Layer

### Responsibility:

-   Data persistence

``` csharp
public class PaymentRepository
{
    public void Save(Payment payment);
    public Payment GetById(string paymentId);
}

public class TransactionRepository
{
    public void Save(PaymentTransaction txn);
}
```

------------------------------------------------------------------------

# 🔁 Idempotency Strategy

-   API → IdempotencyKey
-   Processing → PaymentId
-   Gateway → PaymentId
-   Ledger → PaymentId

------------------------------------------------------------------------

# 🔄 Flow Diagram

``` mermaid
flowchart TD
A[Client] --> B[Idempotency Check]
B --> C[Create Payment]
C --> D[Process Payment]
D -->|Success| E[Ledger Update]
D -->|Fail| F[Retry Scheduler]
F --> D
```

------------------------------------------------------------------------

# ⚠️ Failure Case

``` mermaid
sequenceDiagram
Client->>Service: Request
Service->>Gateway: Charge
Gateway-->>Service: SUCCESS
Service-->>DB: Update (fails)
Retry->>Gateway: GetStatus
Gateway-->>Retry: SUCCESS
Retry->>DB: Update only
```

------------------------------------------------------------------------

# 📦 Cache Usage

-   Redis for Idempotency
-   Optional status cache

------------------------------------------------------------------------

# 🧠 Design Patterns

-   Strategy
-   Factory
-   State
-   Repository

------------------------------------------------------------------------

# ⚙️ NFR Handling

  NFR           Solution
  ------------- ----------------------
  Consistency   Idempotency + Ledger
  Reliability   Retry
  Scalability   Stateless
  Latency       Cache
  Audit         Transactions

------------------------------------------------------------------------

# 🎯 Summary

-   No double charge
-   Safe retries
-   Strong consistency
-   Extensible
