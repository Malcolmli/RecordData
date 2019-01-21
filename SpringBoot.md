# SpringBoot

<!-- TOC -->
 - [Testing单元测试](#Testing单元测试)
     - [Junit](#Junit)
     - [MockMvc](#MockMvc)
     - [Mockito](#Mockito)
 - [NoSql](#NoSql)
<!-- /TOC -->


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

###### RedisTemplate 发送消息
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
    

###### RedisCacheManager
> spring.cache.type配置的为Redis，Spring Boot会自动生成RedisCacheManager对象。

### MongoDB
#### MongoTemplate
> 集成MongoDB  

### Elasticsearch
 
