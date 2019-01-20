
# Spring Boot

## Testing 单元测试

### Junit
> Assert 定义想测试的条件，当条件成立时，assert方法保持沉默，否则抛出异常。  
> Suite 将测试类归成一组  
> Runner 运行测试  

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
### MongoDB
### Elasticsearch
 
