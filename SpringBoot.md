# SpringBoot

- [Testing单元测试](#Testing单元测试)
    - [Junit](#Junit)
    - [MockMvc](#MockMvc)
    - [Mockito](#Mockito)
- [NoSql](#NoSql)
    - [Redis](#Redis)
        - [Jedis](#Jedis)
        - [RedisTemplate](#RedisTemplate)
        - [Redis高级用法](#Redis高级用法)
            - [Redis消息监听器](#Redis消息监听器)
            - [定义Redis的监听容器](#定义Redis的监听容器)
            - [RedisTemplate发送消息](#RedisTemplate发送消息)
        - [Redis缓存](#Redis缓存)
            - [Spring缓存注解](#Spring缓存注解)
            - [RedisCacheManager](#RedisCacheManager)
    - [MongoDB](#MongoDB)
        - [MongoTemplate](#MongoTemplate)
            - [新增](#新增)
            - [查询](#查询)
            - [删除](#删除)
            - [更新](#更新)
    - [Elasticsearch](#Elasticsearch)
- [异步消息队列](#异步消息队列)
    - [JMS实例_ActiveMQ](#JMS实例_ActiveMQ)
    - [AMQP_RabbitMQ](#AMQP_RabbitMQ)

## Testing单元测试

### Junit
>- Assert 定义想测试的条件，当条件成立时，assert方法保持沉默，否则抛出异常。  
>- Suite 将测试类归成一组  
>- Runner 运行测试  

### MockMvc 
> 是Spring提供的专门用于测试Spring MVC 类  

### Mockito
> @MockBean 自动注入，提供模拟实现 或者 mock 创建对象
> when().thenReturn()  
> verify 校验对象的调用  
> InOrder 校验调用顺序 

#### 代码示例
    @RunWith(SpringRunner.class)
    @SpringBootTest(classes = Application.class)
    public class UserServiceTest {
	    @Autowired
	    UserService userService;
	
        @MockBean
	    private CreditSystemService creditSystemService;

	    @Test
	    public void testService() {
		int userId = 10;
		int expectedCredit = 100;
		when(this.creditSystemService.getUserCredit(anyInt())).thenReturn(expectedCredit);
		int credit = userService.getCredit(10);
		assertEquals(expectedCredit, credit);
	    }
    }
    
[完整引用](https://github.com/Malcolmli/SpringBoot2Samples/tree/master/09_test/ch9.test)

## NoSql

### Redis

SpringBoot的starter引用包  

     <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-data-redis</artifactId>
     </dependency>

#### Jedis
> Java的Redis连接驱动 

    // 获取底层Jedis连接
    Jedis jedis = (Jedis) RedisTemplate.getConnectionFactory().getConnection().getNativeConnection();  

#### RedisTemplate
> 会自动从RedisConnectionFactory工厂中获取连接，然后执行对应的Redis命令，在最后还会关闭Redis的连接。  

    //配置RedisTemplate初始化
    public RedisTemplate<Object, Object> initRedisTemplate(RedisConnectionFactory redisConnectionFactory){ }
	//自动注入
	@Autowired
	private RedisTemplate redisTemplate = null;
     
    // 获取字符串操作接口
    redisTemplate.opsForValue();
    // 获取散列操作接口
    redisTemplate.opsForHash();
    // 获取列表操作接口
    redisTemplate.opsForList();
    // 获取集合操作接口
    redisTemplate.opsForSet();
    // 获取有序集合操作接口
    redisTemplate.opsForZSet();
    // 获取基数操作接口
    redisTemplate.opsForHyperLogLog();
    // 获取地理位置操作接口
    redisTemplate.opsForGeo();
    
> 配置对应的序列化器，默认为JdkSerializationRedisSerializer  

    //设置默认字符串序列化器，这样Spring就会把Redis当作字符串处理了
    RedisSerializer stringRedisSerializer = redisTemplate.getStringSerializer();
    redisTemplate.setDefaultSerializer(stringRedisSerializer); 

> SessionCallback和RedisCallback  

它们的作用是让RedisTemplate进行回调，通过它们可以在同一条连接下执行多个Redis命令。其中SessionCallback提供了良好的封装，对于开发者比较友好，因此在实际的开发中应该优先选择使用它；相对而言，RedisCallback接口比较底层，需要处理的内容也比较多，可读性较差，所以在非必要的时候尽量不选择使用它。  

    public void useRedisCallback(RedisTemplate redisTemplate) {
        redisTemplate.execute((RedisConnection rc) -> {
            rc.set("key1".getBytes(), "value1".getBytes());
            rc.hSet("hash".getBytes(), "field".getBytes(), "hvalue".getBytes());
            return null;
        });
    }
    public void useSessionCallback(RedisTemplate redisTemplate) {
        redisTemplate.execute((RedisOperations ro) -> {
            ro.opsForValue().set("key1", "value1");
            ro.opsForHash().put("hash", "field", "hvalue");
        return null;
        });
    }

它们都能够使得RedisTemplate使用同一条Redis连接进行回调，从而可以在同一条Redis连接下执行多个方法，避免RedisTemplate多次获取不同的连接。  

#### Redis高级用法
##### Redis事务
> Redis是支持一定事务能力的NoSQL，在Redis中使用事务，通常的命令组合是watch...multi...exec，也就是要在一个Redis连接中执行多个命令，这时我们可以考虑使用SessionCallback接口来达到这个目的。  

- watch命令是可以监控Redis的一些键
- multi命令是开始事务，开始事务后，该客户端的命令不会马上被执行，而是存放在一个队列里
- exe命令的意义在于执行事务，只是它在队列命令执行前会判断被watch监控的Redis的键的数据是否发生过变化（即使赋予与之前相同的值也会被认为是变化过），如果它认为发生了变化，那么Redis就会取消事务，否则就会执行事务  

> Redis在执行事务时，要么全部执行，要么全部不执行，而且不会被其他客户端打断，这样就保证了Redis事务下数据的一致性。  

    redisTemplate.opsForValue().set("key1", "value1");     
    List list = (List)redisTemplate.execute((RedisOperations operations) -> {
        // 设置要监控key1
        operations.watch("key1");
        // 开启事务，在exec命令执行前，全部都只是进入队列
        operations.multi();
        operations.opsForValue().set("key2", "value2");
        // operations.opsForValue().increment("key1", 1);// 
        // 获取值将为null，因为redis只是把命令放入队列
        Object value2 = operations.opsForValue().get("key2");
        System.out.println("命令在队列，所以value为null【"+ value2 +"】");
        operations.opsForValue().set("key3", "value3");
        Object value3 = operations.opsForValue().get("key3");
        System.out.println("命令在队列，所以value为null【"+ value3 +"】");
        // 执行exec命令，将先判别key1是否在监控后被修改过，如果是则不执行事务，否则就执行事务
        return operations.exec();// 
    }

##### Redis流水线
> 在默认的情况下，Redis客户端是一条条命令发送给Redis服务器的，这样显然性能不高。在关系数据库中我们可以使用批量，也就是只有需要执行SQL时，才一次性地发送所有的SQL去执行，这样性能就提高了许多。对于Redis也是可以的，这便是流水线（pipline）技术，在很多情况下并不是Redis性能不佳，而是网络传输的速度造成瓶颈，使用流水线后就可以大幅度地在需要执行很多命令时提升Redis的性能。  

下面我们使用Redis流水线技术测试10万次读写的功能：  

    Long start = System.currentTimeMillis();
    List list = (List)redisTemplate.executePipelined((RedisOperations operations) -> {
        for (int i=1; i<=100000; i++) {
        	//写操作
            operations.opsForValue().set("pipeline_" + i, "value_" + i);
            //读操作 命令只是进入队列，所以值为空
            String value = (String) operations.opsForValue().get("pipeline_" + i);
        }
        return null;
    });
    Long end = System.currentTimeMillis();
    System.out.println("耗时：" + (end - start) + "毫秒。");

##### Redis发布订阅

###### Redis消息监听器
> 为了接收Redis渠道发送过来的消息，先定义一个消息监听器（MessageListener）

    public class RedisMessageListener implements MessageListener {
    @Override
        public void onMessage(Message message, byte[] pattern) {
            // 消息体
            String body = new String(message.getBody());
            // 渠道名称
            String topic = new String(pattern); 
            System.out.println(body);
            System.out.println(topic);
        }
    }

###### 定义Redis的监听容器
> 定义了一个任务池，并设置了任务池大小为20，这样它将可以运行线程，并进行阻塞，等待Redis消息的传入。接着再定义了一个Redis消息监听的容器RedisMessageListenerContainer，并且往容器设置了Redis连接工厂和指定运行消息的线程池，定义了接收“topic1”渠道的消息，这样系统就可以监听Redis关于“topic1”渠道的消息了。  

    public RedisMessageListenerContainer initRedisContainer() {
        RedisMessageListenerContainer container 
            = new RedisMessageListenerContainer();
        // Redis连接工厂
        container.setConnectionFactory(connectionFactory);
        // 设置运行任务池
        container.setTaskExecutor(initTaskScheduler());
        // 定义监听渠道，名称为topic1
        Topic topic = new ChannelTopic("topic1");
        // 使用监听器监听Redis的消息
        container.addMessageListener(redisMsgListener, topic);
        return container;
    }

###### RedisTemplate发送消息
在Redis的客户端输入命令:  

    publish topic1 msg 

在Spring中，也可以使用RedisTemplate来发送消息:

    redisTemplate.convertAndSend(channel, message);

##### Redis缓存
> Spring在使用缓存注解前，需要配置缓存管理器，缓存管理器将提供一些重要的信息，如缓存类型、超时时间等。Spring可以支持多种缓存的使用，因此它存在多种缓存处理器，并提供了缓存处理器的接口CacheManager和与之相关的类。

    #缓存配置
    spring.cache.type=REDIS # 缓存类型，在默认的情况下，Spring会自动根据上下文探测
    spring.cache.cache-names=redisCache # 如果由底层的缓存管理器支持创建，以逗号分隔的列表来缓存名称
    
    spring.cache.redis.cache-null-values=true # 是否允许Redis缓存空值
    spring.cache.redis.key-prefix= # Redis的键前缀
    spring.cache.redis.time-to-live=0ms # 缓存超时时间戳，配置为0则不设置超时时间
    spring.cache.redis.use-key-prefix=true # 是否启用Redis的键前缀

###### Spring缓存注解
首先是注解@CachePut、@Cacheable和@CacheEvict：
- @CachePut表示将方法结果返回存放到缓存中。
- @Cacheable 表示先从缓存中通过定义的键查询，如果可以查询到数据，则返回，否则执行该方法，返回数据，并且将返回结果保存到缓存中。
- @CacheEvict 通过定义的键移除缓存，它有一个Boolean类型的配置项beforeInvocation，表示在方法之前或者之后移除缓存。因为其默认值为false，所以默认为方法之后将缓存移除。 

###### RedisCacheManager
> spring.cache.type配置的为Redis，Spring Boot会自动生成RedisCacheManager对象。  

### MongoDB
> 对于那些需要统计、按条件查询和分析的数据，MongoDB提供了支持，它可以说是一个最接近于关系数据库的NoSQL。

#### MongoTemplate
> SpringBoot集成MongoDB的操作类  

##### 新增

    // 使用名称为user文档保存用户信息
    mongoTmpl.save(user, "user");
    // 如果文档采用类名首字符小写，则可以这样保存
    mongoTmpl.save(user);

##### 查询
准则（Criteria）构建查询条件  

    // 将用户名称和备注设置为模糊查询准则
    Criteria  criteriaId  = Criteria.where("id").is(id);
    Criteria  criteria = Criteria.where("userName").regex(userName).and("note").regex(note);
    // 构建查询条件,并设置分页跳过前skip个，至多返回limit个
    Query query = Query.query(criteria).limit(limit).skip(skip);

使用find方法，将结果查询为一个列表，返回给调用者
    
    // 执行
	List<User> userList = mongoTmpl.find(query, User.class);

##### 删除
 
采用remove方法将数据删除，执行删除后会返回一个DeleteResult对象来记录此次操作的结果。  

    DeleteResult result = mongoTmpl.remove(query, User.class);

##### 更新

    // 确定要更新的对象
    Criteria criteriaId  = Criteria.where("id").is(id);
    Query query = Query.query(criteriaId);
    // 定义更新对象，后续可变化的字符串代表排除在外的属性
    Update update = Update.update("userName", userName);
    update.set("note", note);
    // 更新第一个文档
    UpdateResult result = mongoTmpl.updateFirst(query, update, User.class);
    // 更新多个对象
    UpdateResult result = mongoTmpl.updateMulti(query, update, User.class);

执行更新方法后，会返回一个UpdateResult对象，它有3个属性，分别是matchedCount、modifiedCount和upsertedId，其中，matchedCount代表与Query对象匹配的文档数，modifiedCount代表被更新的文档数，upsertedId表示如果存在因为更新而插入文档的情况会返回插入文档的信息。

### Elasticsearch

## 异步消息队列
> 发布订阅模式是一个系统约定将消息发布到一个主题（Topic）中，然后各个系统就能够通过订阅这个主题，根据发送过来的信息处理对应的业务。

在实际工作中实现JMS服务的规范有很多，其中比较常用的有传统的ActiveMQ和分布式的Kafka。为了更为可靠和安全，还存在AMQP协议（Advanced Message Queuing Protocol），实现它比较常用的有RabbitMQ等。

### JMS实例_ActiveMQ
> 消息的发送或者接收可以通过模板对象JmsTemplate去处理，关于接收信息，Spring4.1之后的版本提供注解@JmsListener进一步地简化了开发者的工作。
 
    jmsTemplate.convertAndSend(message);

    @JmsListener(destination = "监听地址")

 在默认的情况下，JmsTemplate会提供一个SimpleMessageConverter去提供转换规则，它实现了MessageConverter接口。如果要使用其他的序列化器，如SerializerMessageConverter（序列化消息转换器）或者Jackson2JsonMessageConverter（Json消息转换器），只需要使用JmsTemplate的setMessageConverter方法进行设置即可，不过在一般情况下SimpleMessageConverter已经足够使用了。

### AMQP_RabbitMQ
> AMQP也是一种常用的消息协议。AMQP是一个提供统一消息服务的应用层标准协议，基于此协议的客户端与消息中间件可传递消息，并不受客户端/中间件不同产品、不同开发语言等条件的限制。

创建队列
> Spring Boot的机制会自动注册这两个队列，所以并不需要自己做进一步的绑定。  
    
    @Bean
	public Queue createQueueMsg() {
		// 创建字符串消息队列，boolean值代表是否持久化消息
		return new Queue(“队列名称”, true);
	}

RabbitMQ服务实现

    public class RabbitMqServiceImpl 
        // 实现ConfirmCallback接口，这样可以回调
        implements ConfirmCallback, RabbitMqService {
        @Autowired
        private RabbitTemplate rabbitTemplate = null;
        // 发送消息
        @Override
        public void sendMsg(String msg) {
            System.out.println("发送消息: 【" + msg + "】");
            // 设置回调
            rabbitTemplate.setConfirmCallback(this);
            // 发送消息，通过msgRouting确定队列
            rabbitTemplate.convertAndSend(msgRouting, msg);
        }
        // 回调确认方法
        @Override
        public void confirm(CorrelationData correlationData, 
            boolean ack, String cause) {
            if (ack) {
                System.out.println("消息成功消费");
            } else {
                System.out.println("消息消费失败:" + cause);
            }
        }
    }

RabbitMQ接收器
> 在方法上标注@RabbitListener，然后在其配置项queues配置所需要的队列名称，这样就能够直接接收到RabbitMQ所发送的消息。

    // 定义监听字符串队列名称
    @RabbitListener(queues = { "${rabbitmq.queue.msg}" })
    public void receiveMsg(String msg) {
         System.out.println("收到消息: 【" + msg + "】");
    }