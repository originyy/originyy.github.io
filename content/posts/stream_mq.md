---
title: "Stream_mq"
date: 2023-03-12T03:22:17+08:00
draft: true
---


由于Spring Cloud Stream的注解已过期@Deprecated ，需要升级到更高版本。并且目前目前mq没有做到可靠消费，在服务中断等特殊情况会把消息丢失。现需要对Spring Cloud Stream和MQ进行升级。  

写文档时GA版本有4.0.1和3.2.4：  

4.0.1遇到了一个较为麻烦的问题，疑似Bean加载顺序有问题，具体需要深入排查。  

所以选择了3.2.4。  

pom依赖：  

	<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-stream-rabbit</artifactId>
	<version>3.2.4</version>
	</dependency>
	<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-stream</artifactId>
	<version>3.2.4</version>
	</dependency>

3.2.4也有一个bug，会有一处地方直接传null，详情可看[Spring cloud stream 3.2.3 FunctionInvocationHelper throw NPE Issue #2384 · spring-cloud/spring-cloud-stream (github.com)](https://github.com/spring-cloud/spring-cloud-stream/issues/2384)  

需要替换function依赖。  

	<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-function-context</artifactId>
	<version>3.2.4</version>
	</dependency>

旧配置@EnableBinding 会和函数式的写法冲突，需要去除，所以升级后不再兼容旧写法。  

发布者需要手动发布，同时用confirm实现可靠发布  

	@Service
	@RequiredArgsConstructor
	@Slf4j
	public class EventPublisher {
	 
	 
		private final StreamBridge streamBridge;
	 
		/**
		 * @param event
		 * @return void
		 * @throws
		 * @description
		 * @author yang
		 * @date 2022/7/12 11:41
		 */
		public void pushEvent(String channel, Object event) {
			MyCorrelationData corr = new MyCorrelationData(UUID.randomUUID().toString(), event);
			streamBridge.send(channel, MessageBuilder.withPayload(event)
					.setHeader(AmqpHeaders.PUBLISH_CONFIRM_CORRELATION, corr).build());
	 
			try {
				CorrelationData.Confirm confirm = corr.getFuture().get(10, TimeUnit.SECONDS);
				log.info(confirm + " for " + corr.getPayload());
	 
				if (Boolean.FALSE.equals(confirm.isAck())) {
					log.error("Message for " + corr.getPayload() + " was returned ,reason is :" + confirm.getReason());
					throw new BadRequestException(HttpStatus.BAD_REQUEST,"事件发送失败");
				}
			} catch (InterruptedException e) {
				Thread.currentThread().interrupt();
				throw new IllegalStateException(e);
			} catch (ExecutionException | TimeoutException e) {
				throw new IllegalStateException(e);
			}
		}
	}
也可以采用队列+Supplier，这里不多做介绍。  


消费者改为用函数式并注册为bean，同时实现可靠消费  

	@Bean
	public Consumer<Message<StudentEvent>> changeStudentStatus() {
		return m ->
		{
			Channel channel = m.getHeaders().get(AmqpHeaders.CHANNEL, Channel.class);
			Long deliveryTag = m.getHeaders().get(AmqpHeaders.DELIVERY_TAG, Long.class);
	 
				//业务逻辑处理
				StudentEvent event = m.getPayload();
				customerFacade.changeStudentStatusByStudentEvent(event);
			try {
				//ack确认
				channel.basicAck(deliveryTag, false);
			} catch (IOException e) {
				log.error(e.getMessage());
			}
		};
	}

同时RabbitMQ需要设置 durable = true，一般默认是true，保证MQ持久化队列中的消息。  

yml配置样例：  

	rabbitmq:
		port: 5672
		host: 192.168.1.166
		username: guest
		password: guest
		publisher-confirm-type: correlated  # confirm
		publisher-returns: true # confirm
	 
	  cloud:
		function:
		  # 类似组合的定义 enrich|echo  ，enrich|echo整体就是一个functionName
			definition: createLead;generateLeadNotice;
		stream:
		  source: leadReceive;leadEvent;
		  bindings:
			createLead-in-0:
			  destination: lead_receive_test   #exchange名称，交换模式默认是topic
			  content-type: application/json
			  binder: rabbit
			  group: createLead-in-0
			  consumer:
				concurrency: 1
				max-attempts: 3
			generateLeadNotice-in-0:
			  destination: lead_event_test   #exchange名称，交换模式默认是topic
			  content-type: application/json
			  binder: rabbit
			  group: generateLeadNotice-in-0
			  consumer:
				concurrency: 1
				max-attempts: 3
			leadReceive-out-0:
			  destination: lead_receive_test   #exchange名称，交换模式默认是topic
			  content-type: application/json
			  binder: rabbit
			  producer:
				error-channel-enabled: true
			  consumer:
				concurrency: 1
				max-attempts: 1
			leadEvent-out-0:
			  destination: lead_event_test   #exchange名称，交换模式默认是topic
			  content-type: application/json
			  binder: rabbit
			  producer:
				error-channel-enabled: true
			  consumer:
				concurrency: 1
				max-attempts: 1
		  rabbit:
			bindings:
			  createLead-in-0:
				consumer:
				  acknowledge-mode: MANUAL # 手动确认
			  generateLeadNotice-in-0:
				consumer:
				  acknowledge-mode: MANUAL
			  checkEvaluation-out-0:
				producer:
				  delayedExchange: true
				  useConfirmHeader: true
			  leadReceive-out-0:
				producer:
				  useConfirmHeader: true
			  leadEvent-out-0:
				producer:
				  useConfirmHeader: true
更多使用和配置可查看官方文档：Spring Cloud Stream