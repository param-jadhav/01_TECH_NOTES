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
