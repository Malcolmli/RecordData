
# Spring Boot

## Testing 单元测试

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

#### RedisTemplate
> 会自动从RedisConnectionFactory工厂中获取连接，然后执行对应的Redis命令，在最后还会关闭Redis的连接。  
   
> 配置对应的序列化器，默认为JdkSerializationRedisSerializer  

    //配置RedisTemplate初始化
    public RedisTemplate<Object, Object> initRedisTemplate(RedisConnectionFactory redisConnectionFactory){ }
	//自动注入
	@Autowired
	private RedisTemplate redisTemplate = null;

> SessionCallback和RedisCallback  

它们的作用是让RedisTemplate进行回调，通过它们可以在同一条连接下执行多个Redis命令。其中SessionCallback提供了良好的封装，对于开发者比较友好，因此在实际的开发中应该优先选择使用它；相对而言，RedisCallback接口比较底层，需要处理的内容也比较多，可读性较差，所以在非必要的时候尽量不选择使用它。  

### MongoDB
#### MongoTemplate
> 集成MongoDB  

### Elasticsearch
 
