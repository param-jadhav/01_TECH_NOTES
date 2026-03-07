# 01_TECH_NOTES

#01 Kafka Messaging
-------------------
@RestController
@RequestMapping("/data")
public class MyRestController {

    @Autowired
    private KafkaTemplate<String, MyDataModel> kafkaTemplate;  // Assuming MyDataModel is your data class

    @PostMapping("/push")
    public ResponseEntity<String> pushData(@RequestBody MyDataModel data) {
        // 1. Send data to Kafka asynchronously using CompletableFuture
        CompletableFuture<SendResult<String, MyDataModel>> future = kafkaTemplate.send("yourTopic", data);

        // 2. Handle successful or failed message sending (optional)
        future.whenComplete((result, ex) -> {
            if (ex != null) {
                System.err.println("Error sending message: " + ex.getMessage());
                // Optionally, return an error response to the client
                return;
            }
            // Message sent successfully, perform post-processing if needed (e.g., logging)
        });

        // 3. Return immediate response to client (doesn't wait for Kafka ack)
        return ResponseEntity.ok("Data pushed to Kafka successfully");
    }
}


@Service
public class KafkaService {

    @Autowired
    private KafkaTemplate<String, KafkaRequest> kafkaTemplate;

    public KafkaResponse sendDataToKafka(ProducerRequest kafkaProducerRequest) {
        KafkaRequest kafkaRequest = new KafkaRequest();
        KafkaResponse kafkaResponse = new KafkaResponse();
        BeanUtils.copyProperties(kafkaProducerRequest, kafkaRequest);
        kafkaRequest.setApplicationId(kafkaProducerRequest.getApplicationIdWithPrefix());
        
        try {
            SendResult<String, KafkaRequest> result = kafkaTemplate.send(kafkaRequest.getTopicName(), kafkaRequest).get();
            
            MDC.put(ServiceConstants.TRANSACTION_ID, result.getProducerRecord().value().getTransactionId());
            log.info("Successfully published the message to the kafka topic: {}, for the application id: {}, "
                    + "clientId: {}, storeNumber: {}, merchantNumber: {}, messageType: {}, partnerCode: {}, "
                    + "applicationSource: {}, partition: {}, Offset: {}",
                    kafkaProducerRequest.getTopicName(), kafkaRequest.getApplicationId(),
                    kafkaProducerRequest.getClientId(), kafkaProducerRequest.getStoreNumber(),
                    kafkaProducerRequest.getMerchantNumber(), kafkaProducerRequest.getMessageType(),
                    kafkaProducerRequest.getPartnerCode(), kafkaProducerRequest.getApplicationSource(),
                    result.getRecordMetadata().partition(), result.getRecordMetadata().offset());
            
            kafkaResponse.setPartition(result.getRecordMetadata().partition());
            kafkaResponse.setOffset(result.getRecordMetadata().offset());
            kafkaResponse.setDecisionCode("SUCCESS");
            kafkaResponse.setDecisionMessage("Message pushed successfully");
        } catch (Exception ex) {
            MDC.put(ServiceConstants.TRANSACTION_ID, kafkaProducerRequest.getTransactionId());
            log.error("Error Occured while publishing data to Kafka", ex.getMessage());
            kafkaResponse.setPartition(-1);
            kafkaResponse.setOffset((long) -1);
            kafkaResponse.setDecisionCode("ERROR");
            kafkaResponse.setDecisionMessage("Failed to send message: " + ex.getMessage());
        }
        
        return kafkaResponse;
    }
}
In this code:

The sendDataToKafka method in KafkaService uses get() on the Future returned by kafkaTemplate.send() to wait for the send operation to complete.
If the send operation succeeds, it logs the success and sets the relevant fields in KafkaResponse.
If an exception occurs, it logs the error and sets the error details in KafkaResponse.
The controller method calls sendDataToKafka and directly returns the KafkaResponse.
This approach ensures that the Kafka send operation is completed before the controller sends a response to the client. However, be aware that blocking the thread while waiting for Kafka may impact the performance and scalability of your application, especially under heavy load.



.retryWhen(
    Retry.fixedDelay(
        emailConsumerPropertiesConfig.getMaxAttempts(),
        Duration.ofSeconds(emailConsumerPropertiesConfig.getRetryDelaySeconds())
    )
    .filter(throwable -> throwable instanceof WebClientResponseException)
    .doBeforeRetry(r ->
        log.info("Transaction Id: {} | Retrying attempt #{} due to {}",
                 transactionId, r.totalRetries(), r.failure().getMessage()))
)
.doOnSuccess(v -> {
    savedEmailDataDTO.setStatus(ServiceConstants.EMAIL_SUCCESS_STATUS);
    emailDataDao.saveEmailData(savedEmailDataDTO);
    log.info("Transaction Id: {} | Send Email success", transactionId);
})
.doOnError(ex -> {
    // Update DB for failure
    savedEmailDataDTO.setStatus(ServiceConstants.EMAIL_FAILED_STATUS);
    emailDataDao.saveEmailData(savedEmailDataDTO);
    log.error("Transaction Id: {} | Send Email Failed: {}", transactionId, ex.getMessage(), ex);
})
// This is the key part: convert the error so Rabbit can DLQ
.onErrorMap(ex -> new AmqpRejectAndDontRequeueException("Force DLQ after REST failure", ex))
.subscribe();





# Copilot Instructions for Developers

## System Overview
This repository contains a financial services platform that processes credit card applications, account events, and payment transactions.

Key capabilities:
- Credit application processing
- Transaction event streaming
- Payment authorization workflows
- External partner integrations

Tech Stack:
- Java 17
- Spring Boot
- Apache Kafka
- PostgreSQL
- Redis Cache
- REST APIs

Architecture style:
- Event-driven microservices
- Domain-driven design
- Asynchronous messaging

---

# Coding Standards

General guidelines for generated code:

- Follow clean code principles.
- Prefer readability over clever implementations.
- Keep methods small and single responsibility.
- Avoid deep nesting (>3 levels).
- Use descriptive variable and method names.
- Avoid static utility classes when dependency injection is possible.

Java specific rules:
- Use Java Streams only when it improves readability.
- Prefer Optional instead of returning null.
- Use immutable DTOs where possible.

---

# Security Guidelines (Critical)

This is a financial system and must comply with strict security rules.

Never log or expose the following:

- Credit card numbers (PAN)
- CVV codes
- SSN
- Bank account numbers
- Authentication tokens

Rules:
- Mask sensitive data before logging.
- Always validate input payloads.
- Use parameterized SQL queries.
- Never construct dynamic SQL strings.
- Use encryption for sensitive data at rest and in transit.

---

# Logging Standards

Logging must support traceability and auditing.

Use structured logging with these fields:

- traceId
- correlationId
- transactionId
- serviceName

Rules:
- Never log PII or financial details.
- Log external API failures with status codes.
- Include correlationId in all Kafka messages.

Example log format:

INFO PaymentAuthorized traceId=abc123 correlationId=txn456 amount=200

---

# Exception Handling

All services must use centralized exception handling.

Rules:
- Use GlobalExceptionHandler.
- Do not expose internal stack traces to API clients.
- Convert exceptions into standardized error responses.

Standard API error format:

{
  "errorCode": "PAYMENT_DECLINED",
  "message": "Payment authorization failed",
  "traceId": "abc123"
}

---

# Kafka Event Guidelines

All services communicate via Kafka events.

Rules:
- Events must be immutable.
- Include schema version.
- Use clear event naming.

Standard event fields:

- eventId
- eventType
- eventVersion
- timestamp
- correlationId
- payload

Topic naming standard:

domain.service.event

Example:

credit.application.submitted

---

# Database Standards

Database: PostgreSQL

Rules:
- Use Flyway for schema migrations.
- Avoid SELECT * queries.
- Always specify columns explicitly.
- Add indexes for frequently queried fields.
- Use optimistic locking for financial records.

Transactions:
- Use transactional boundaries at service layer.

---

# REST API Standards

Follow REST best practices.

Rules:
- Use correct HTTP status codes.
- Version APIs when making breaking changes.
- Validate request payloads using annotations.

Example response structure:

{
  "status": "SUCCESS",
  "data": {},
  "errors": []
}

---

# Testing Guidelines

Unit Testing:
- Use JUnit 5
- Use Mockito for mocking
- Cover business logic and edge cases

Integration Testing:
- Test Kafka consumers and producers
- Test database repositories
- Validate event schema compatibility

Target coverage: minimum 80%.

---

# Performance Guidelines

Financial systems must support high transaction volumes.

Rules:
- Avoid synchronous blocking calls in event consumers.
- Use caching for frequently accessed reference data.
- Batch database operations when possible.
- Avoid unnecessary object creation in loops.

---

# AI Code Generation Expectations

When generating code, Copilot should:

- Follow existing architecture patterns
- Use constructor injection
- Write production-ready code
- Include meaningful logging
- Handle errors gracefully
- Follow domain naming conventions

Avoid:
- Placeholder logic
- insecure implementations
- hardcoded values
- missing validation