# Durable Execution, Workflows, và Event-Driven Architectures

---

## 1. Workflows — Chuỗi Service Calls

### Vấn đề trong Service-Based Architectures

```
Service-based architecture: nhiều services, mỗi cái chịu trách nhiệm
một PHẦN của application

Ví dụ: Payment Processing
   Một payment cần: fraud detection → credit card charge → bank deposit

→ Processing 1 payment = nhiều service calls = WORKFLOW
   Mỗi bước = TASK
```

### Workflow Engines

```
Workflow: định nghĩa như GRAPH of tasks
   (code, DSL, hoặc markup như BPEL)

Workflow engine quyết định:
   - Khi nào / trên machine nào run từng task
   - Phải làm gì nếu task FAIL
   - Bao nhiêu tasks được run song song

2 components:
   ORCHESTRATOR: schedule tasks
   EXECUTOR: thực sự execute tasks

Trigger: time-based schedule (hourly), external source (web service, human)
```

### Loại Workflow Engines

| Loại                   | Ví dụ                     | Focus                          |
| ---------------------- | ------------------------- | ------------------------------ |
| Data/ETL orchestration | Airflow, Dagster, Prefect | Integrate với data systems     |
| Graphical (BPMN)       | Camunda, Orkes            | Non-engineers define workflows |
| Durable execution      | Temporal, Restate         | Exactly-once semantics         |

---

## 2. Durable Execution

### Vấn đề: Partial Failure

```
Payment: Fraud check OK → Credit card charged → CRASH → bank deposit KHÔNG xảy ra
→ Khách hàng bị trừ tiền nhưng KHÔNG nhận được dịch vụ!

KHÔNG THỂ wrap trong database transaction:
   Service calls là EXTERNAL → outside transaction boundary
```

### Giải pháp: Log mọi thứ vào Durable Storage

```
Durable execution framework LOG tất cả RPCs và state changes
   vào DURABLE STORAGE (giống WAL)

Nếu task FAIL:
   Framework RE-EXECUTE task, nhưng SKIP calls đã thành công
   → "Pretend" thực hiện call → trả kết quả từ lần trước

→ EXACTLY-ONCE SEMANTICS cho toàn bộ workflow
```

### Ví dụ: Temporal (Python)

```python
@workflow.defn
class PaymentWorkflow:
    @workflow.run
    async def run(self, payment: PaymentRequest) -> PaymentResult:
        is_fraud = await workflow.execute_activity(
            check_fraud, payment,
            start_to_close_timeout=timedelta(seconds=15),
        )
        if is_fraud:
            return PaymentResultFraudulent

        credit_card_response = await workflow.execute_activity(
            debit_credit_card, payment,
            start_to_close_timeout=timedelta(seconds=15),
        )
        # ...
```

### 3 Thách thức Chính

**1. External services phải Idempotent:**

```
External payment gateway: framework không control
→ Phải dùng unique IDs để prevent duplicate execution
```

**2. Code Changes Brittle:**

```
Framework expect same RPC calls theo CÙNG THỨ TỰ mỗi lần replay
→ Re-ordering function calls = UNDEFINED BEHAVIOR

An toàn hơn: deploy NEW VERSION code riêng biệt
   Old workflow invocations → dùng old code
   New invocations → dùng new code
```

**3. Code phải Deterministic:**

```
Durable frameworks replay code deterministically
(cùng inputs → cùng outputs)

Nondeterministic code gây vấn đề:
   - Random number generators
   - System clocks

Frameworks cung cấp deterministic implementations → phải NHỚ dùng
Static analysis tools: Temporal's Workflow Check
```

> Making code deterministic = powerful nhưng tricky. Quay lại ở Chapter 9.

---

## 3. Event-Driven Architectures

### RPC vs Event-Driven

```
RPC: sender GỬI và CHỜ (synchronous)

EVENT-DRIVEN:
   Request = EVENT / MESSAGE
   Sender KHÔNG CHỜ (asynchronous)
   Message đi qua MESSAGE BROKER (intermediary)
   Broker LƯU message tạm thời
```

### 5 Lợi ích của Message Broker

| Lợi ích                         | Giải thích                                            |
| ------------------------------- | ----------------------------------------------------- |
| **Buffer**                      | Recipient unavailable/overloaded → broker giữ message |
| **Redeliver sau crash**         | Tự động redeliver → messages không bị mất             |
| **Không cần service discovery** | Sender không cần biết IP của recipient                |
| **Fan-out**                     | 1 message → nhiều recipients                          |
| **Decoupling**                  | Sender chỉ publish, không cần biết ai consume         |

---

## 4. Message Brokers — Chi tiết

### Lịch sử và Ví dụ

```
Commercial: TIBCO, IBM WebSphere, webMethods
Open source: RabbitMQ, ActiveMQ, HornetQ, NATS, Redpanda, Apache Kafka
Cloud: Amazon Kinesis, Azure Service Bus, Google Cloud Pub/Sub
```

### 2 Distribution Patterns

**Queue (Point-to-Point):**

```
1 producer → NAMED QUEUE → 1 trong số nhiều consumers nhận
```

**Topic (Publish-Subscribe):**

```
1 publisher → NAMED TOPIC → TẤT CẢ subscribers đều nhận
```

### Encoding và Schema Registry

```
Message brokers: KHÔNG enforce data model cụ thể
Message = sequence of bytes + metadata

Encoding phổ biến: Protocol Buffers, Avro, JSON

Best practice: Schema Registry cùng với message broker
   (Confluent Schema Registry, Red Hat Apicurio)
   → Lưu valid schema versions + check compatibility

AsyncAPI: messaging-based equivalent của OpenAPI
```

### Durability

```
Nhiều brokers ghi messages vào DISK
Khác databases: thường AUTO-DELETE sau khi consumed

Configured để store indefinitely:
   → Cần cho EVENT SOURCING (Chapter 3, 12)

Cảnh báo khi republish messages: preserve unknown fields!
   (Tránh vấn đề như Figure 5-1 — code cũ mất data của code mới)
```

---

## 5. Distributed Actor Frameworks

### Actor Model (Single Process)

```
Actor: đơn vị concurrency
   - LOCAL STATE (không share)
   - Communicate qua ASYNC MESSAGES
   - Xử lý 1 message / lần
   - Không lo thread safety

Message delivery KHÔNG guaranteed
→ Actors được thiết kế để chịu được điều này
```

### Scale lên Distributed

```
Frameworks: Akka, Orleans, Erlang/OTP

Cùng message-passing mechanism dù sender/recipient:
   - Cùng node: in-memory delivery
   - Khác node: encode → network → decode (transparent)

Location transparency HOẠT ĐỘNG TỐT HƠN RPC vì:
   Actor model đã ASSUME message có thể bị mất
   → Không có fundamental mismatch local vs remote

Distributed actor framework = Message Broker + Actor Model (1 framework)

Rolling upgrades: vẫn cần backward/forward compatible encoding!
```

---

## Tóm tắt phần này

```
WORKFLOWS:
   Multi-step service calls = workflow, mỗi bước = task
   Durable execution (Temporal, Restate): exactly-once qua WAL logging
   Challenges: external idempotency | brittle code | determinism

MESSAGE BROKERS:
   Buffer + redeliver + fan-out + decoupling
   Queue (point-to-point) vs Topic (pub-sub)
   Schema Registry: Confluent, Apicurio

ACTOR MODEL:
   Local state + async messages, 1 message/lần
   Distributed: Akka, Orleans, Erlang/OTP
   Location transparency tốt hơn RPC (đã expect message loss)
   Vẫn cần compatible encoding cho rolling upgrades
```
