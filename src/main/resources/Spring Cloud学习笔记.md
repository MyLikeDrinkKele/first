# Spring Cloud学习笔记

## 简介

​	spring cloud，是一种微服务框架，一站式分布式系统解决方案，基于spring boot，五大核心分别是：

+ 服务的注册与发现--Netflix Eureka
+ 客户端负载均衡--Netflix Ribbon
+ 断路器--Netflix Hystrix 
+ 服务网关--Netflix Zuui 
+ 配置管理--Spring Cloud Config 

## 1，服务注册中心Eureka

​	作用是维护所有的服务实例

### 1.1单点服务注册中心

搭建步骤:

1. 创建一个最基础的spring boot工程，加入依赖：

   ~~~xml
   <!--eureka依赖-->
   <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka-server</artifactId>
   </dependency>
   <!--统一版本管理-->
   <dependencyManagement>
       <dependencies>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-dependencies</artifactId>
               <version>Dalston.SR3</version>
               <type>pom</type>
               <scope>import</scope>
           </dependency>
       </dependencies>
   </dependencyManagement>
   ~~~

2. 在入口类上添加注解，表明该项目是一个服务注册中心:

   ~~~java
   @EnableDiscoveryClient
   @SpringBootApplication
   public class ProviderApplication {
   
   	public static void main(String[] args) {
   		SpringApplication.run(ProviderApplication.class, args);
   	}
   }
   ~~~

3. 在配置文件里面，配置服务中心的基本信息

   1.server.port=1111表示设置该服务注册中心的端口号 

   2.eureka.instance.hostname=localhost表示设置该服务注册中心的hostname 

   3.eureka.client.register-with-eureka=false,由于我们目前创建的应用是一个服务注册中心，而不是普通的应用，默认情况下，这个应用会向注册中心（也是它自己）注册它自己，设置为false表示禁止这种默认行为 4.eureka.client.fetch-registry=false,表示不去检索其他的服务，因为服务注册中心本身的职责就是维护服务实例，它也不需要去检索其他服务 

   5.eureka.client.service-url.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka/ 表示服务注册路径

### 1.2集群高可用服务注册中心

​	Eureka Server的高可用实际上就是将自己作为服务向其他服务注册中心注册自己，这样就会形成一组互相注册的服务注册中心，进而实现服务清单的互相同步，达到高可用的效果。 同一个项目，以不同的配置文件启动，且相互注册，即可实现高可用，服务注册时，写多个地址，以逗号隔开。

核心概念：

+ 失效剔除：我们说到了服务下线问题，正常的服务下线发生流程有一个前提那就是服务正常关闭,但是在实际运行中服务有可能没有正常关闭，比如系统故障、网络故障等原因导致服务提供者非正常下线，那么这个时候对于已经下线的服务Eureka采用了定时清除：Eureka Server在启动的时候会创建一个定时任务，每隔60秒就去将当前服务提供者列表中超过90秒还没续约的服务剔除出去，通过这种方式来避免服务消费者调用了一个无效的服务。 
+ 自我保护：Eureka Server在运行期间会去统计心跳失败比例在15分钟之内是否低于85%，如果低于85%，Eureka Server会将这些实例保护起来，让这些实例不会过期，但是在保护期内如果服务刚好这个服务提供者非正常下线了，此时服务消费者就会拿到一个无效的服务实例，此时会调用失败，对于这个问题需要服务消费者端要有一些容错机制，如重试，断路器等。我们在单机测试的时候很容易满足心跳失败比例在15分钟之内低于85%，这个时候就会触发Eureka的保护机制，一旦开启了保护机制，则服务注册中心维护的服务实例就不是那么准确了，此时我们可以使用`eureka.server.enable-self-preservation=false`来关闭保护机制，这样可以确保注册中心中不可用的实例被及时的剔除。 

## 2，服务提供者

​	向服务注册中心注册服务

搭建步骤：

1. 创建项目与依赖与上一步一致

2. 在入口类上添加注解，激活服务端：

   ~~~java
   @EnableDiscoveryClient
   @SpringBootApplication
   public class ProviderApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(ProviderApplication.class, args);
       }
   }
   ~~~

3. 创建服务类

   ~~~java
   @RestController
   public class HelloController {
       private final Logger logger = Logger.getLogger(getClass());
       @Autowired
       private DiscoveryClient client;
   
       @RequestMapping(value = "/hello", method = RequestMethod.GET)
       public String index() {
           //将服务注册到中心
           List<ServiceInstance> instances = client.getInstances("hello-service");
           for (int i = 0; i < instances.size(); i++) {
               logger.info("/hello,host:" + instances.get(i).getHost() + ",service_id:" + instances.get(i).getServiceId());
           }
           return "Hello World";
       }
   }
   ~~~

4. 配置

   ~~~java
   spring.application.name=hello-service
   eureka.client.service-url.defaultZone=http://localhost:1111/eureka
   ~~~

核心概念：

+ 服务注册：服务提供者在启动的时候会通过发送REST请求将自己注册到Eureka Server上，同时还携带了自身服务的一些元数据信息。Eureka Server在接收到这个REST请求之后，将元数据信息存储在一个双层结构的Map集合中，第一层的key是服务名，第二层的key是具体服务的实例名 。在服务注册时，需要确认一下`eureka.client.register-with-eureka=true`配置是否正确，该值默认就为true，表示启动注册操作，如果设置为false则不会启动注册操作。 

+ 服务同步：假如两个服务提供者的信息分别被两个服务注册中心所维护，但是由于服务注册中心之间也互相注册为服务，当服务提供者发送请求到一个服务注册中心时，它会将该请求转发给集群中相连的其他注册中心，从而实现注册中心之间的服务同步，通过服务同步，两个服务提供者的服务信息我们就可以通过任意一台注册中心来获取到。 

+ 服务续约：在注册完服务之后，服务提供者会维护一个心跳来不停的告诉Eureka Server：“我还在运行”，以防止Eureka Server将该服务实例从服务列表中剔除，这个动作称之为服务续约，和服务续约相关的属性有两个，如下：

  ```
  eureka.instance.lease-expiration-duration-in-seconds=90  
  eureka.instance.lease-renewal-interval-in-seconds=30
  ```

  第一个配置用来定义服务失效时间，默认为90秒，第二个用来定义服务续约的间隔时间，默认为30秒。

## 3，服务消费者

​	服务的发现和消费实际上是两个行为，这两个行为要由不同的对象来完成：服务的发现由Eureka客户端来完成，而服务的消费由Ribbon来完成。Ribbo是一个基于HTTP和TCP的客户端负载均衡器，当我们将Ribbon和Eureka一起使用时，Ribbon会从Eureka注册中心去获取服务端列表，然后进行轮询访问以到达负载均衡的作用，服务端是否在线这些问题则交由Eureka去维护。 

搭建步骤：

1. 创建一个spring boot项目，started添加web，然后添加Eureka和Ribbon依赖 

   ~~~xml
   		<dependency>
   			<groupId>org.springframework.cloud</groupId>
   			<artifactId>spring-cloud-starter-eureka</artifactId>
   		</dependency>
   
   		<dependency>
   			<groupId>org.springframework.cloud</groupId>
   			<artifactId>spring-cloud-starter-ribbon</artifactId>
   		</dependency>
   ~~~

2. 配置入口类，添加注解，并提供RestTemplate的Bean

   ~~~java
   @SpringBootApplication
   @EnableDiscoveryClient
   public class RibbonConsumerApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(RibbonConsumerApplication.class, args);
       }
       @LoadBalanced
       @Bean
       RestTemplate restTemplate() {
           return new RestTemplate();
       }
   }
   ~~~

3. 创建一个Controller类，并向Controller类中注入RestTemplate对象 

   ~~~java
   @RestController
   public class ConsumerController {
       @Autowired
       RestTemplate restTemplate;
       @RequestMapping(value = "/ribbon-consumer",method = RequestMethod.GET)
       public String helloController() {
           return restTemplate.getForEntity("http://HELLO-SERVICE/hello", String.class).getBody();
       }
   }
   ~~~

4. 配置文件配置服务中心地址

   ~~~javascript
   spring.application.name=ribbon-consumer  
   server.port=9000  
   eureka.client.service-url.defaultZone=http://peer1:1111/eureka
   ~~~

核心概念：

+ 获取服务：当我们启动服务消费者的时候，它会发送一个REST请求给服务注册中心来获取服务注册中心上面的服务提供者列表，而Eureka Server上则会维护一份只读的服务清单来返回给客户端，这个服务清单并不是实时数据，而是一份缓存数据，默认30秒更新一次，如果想要修改清单更新的时间间隔，可以通过`eureka.client.registry-fetch-interval-seconds=30`来修改，单位为秒(注意这个修改是在eureka-server上来修改)。另一方面，我们的服务消费端要确保具有获取服务提供者的能力，此时要确保`eureka.client.fetch-registry=true`这个配置为true。 
+ 服务调用：服务消费者从服务注册中心拿到服务提供者列表之后，通过服务名就可以获取具体提供服务的实例名和该实例的元数据信息，客户端将根据这些信息来决定调用哪个实例，我们之前采用了Ribbon，Ribbon中默认采用轮询的方式去调用服务提供者，进而实现了客户端的负载均衡。 
+ 服务下线：服务提供者在运行的过程中可能会发生关闭或者重启，当服务进行**正常**关闭时，它会触发一个服务下线的REST请求给Eureka Server，告诉服务注册中心我要下线了，服务注册中心收到请求之后，将该服务状态置为DOWN，表示服务已下线，并将该事件传播出去，这样就可以避免服务消费者调用了一个已经下线的服务提供者了。 

## 4，RestTemplate

​	服务消费端去调用服务提供者的服务的时候，使用了一个很好用的对象，叫做RestTemplate ，它可以发起四种不同的请求。

### 4.1  GET请求

**1，第一种：getForEntity**

​	getForEntity方法的返回值是一个`ResponseEntity<T>`，`ResponseEntity<T>`是Spring对HTTP请求响应的封装，包括了几个重要的元素，如响应码、contentType、contentLength、响应消息体等。 

~~~java
@RequestMapping("/gethello")
public String getHello() {
    ResponseEntity<String> responseEntity = restTemplate.getForEntity("http://HELLO-SERVICE/hello", String.class);
    String body = responseEntity.getBody();
    HttpStatus statusCode = responseEntity.getStatusCode();
    int statusCodeValue = responseEntity.getStatusCodeValue();
    HttpHeaders headers = responseEntity.getHeaders();
    StringBuffer result = new StringBuffer();
    result.append("responseEntity.getBody()：").append(body).append("<hr>")
            .append("responseEntity.getStatusCode()：").append(statusCode).append("<hr>")
            .append("responseEntity.getStatusCodeValue()：").append(statusCodeValue).append("<hr>")
            .append("responseEntity.getHeaders()：").append(headers).append("<hr>");
    return result.toString();
}
~~~

- getForEntity的第一个参数为我要调用的服务的地址，这里我调用了服务提供者提供的/hello接口，注意这里是通过服务名调用而不是服务地址，如果写成服务地址就没法实现客户端负载均衡了。
- getForEntity第二个参数String.class表示我希望返回的body类型是String。如果接口返回的是一个自定义对象，则把String.class换成对象字节码即可。

**参数传递**

~~~java
@RequestMapping("/sayhello")
public String sayHello() {
    ResponseEntity<String> responseEntity = restTemplate.getForEntity("http://HELLO-SERVICE/sayhello?name={1}", String.class, "张三");
    return responseEntity.getBody();
}
@RequestMapping("/sayhello2")
public String sayHello2() {
    Map<String, String> map = new HashMap<>();
    map.put("name", "李四");
    ResponseEntity<String> responseEntity = restTemplate.getForEntity("http://HELLO-SERVICE/sayhello?name={name}", String.class, map);
    return responseEntity.getBody();
}
~~~

- 可以用一个数字做占位符，最后是一个可变长度的参数，来一一替换前面的占位符
- 也可以前面使用name={name}这种形式，最后一个参数是一个map，map的key即为前边占位符的名字，map的value为参数值。

**2，第二种：getForObject**

​	getForObject函数实际上是对getForEntity函数的进一步封装，如果你只关注返回的消息体的内容，对其他信息都不关注，此时可以使用getForObject。几个方法重载与getForEntity一致。

### 4.2 POST请求

**第一种：postForEntity**

~~~java
@RequestMapping("/book3")
public Book book3() {
    Book book = new Book();
    book.setName("红楼梦");
    ResponseEntity<Book> responseEntity = restTemplate.postForEntity("http://HELLO-SERVICE/getbook2", book, Book.class);
    return responseEntity.getBody();
}
~~~

- 方法的第一参数表示要调用的服务的地址
- 方法的第二个参数表示上传的参数
- 方法的第三个参数表示返回的消息体的数据类型

**第二种：postForObject**

​	如果你只关注，返回的消息体，可以直接使用postForObject。用法和getForObject一致。 

**第三种：postForLocation**

​	postForLocation也是提交新资源，提交成功之后，返回新资源的URI，postForLocation的参数和前面两种的参数基本一致，只不过该方法的返回值为Uri，这个只需要服务提供者返回一个Uri即可，该Uri表示新资源的位置。

### 4.3 PUT请求

​	在RestTemplate中，PUT请求可以通过put方法调用，put方法的参数和前面介绍的postForEntity方法的参数基本一致，只是put方法没有返回值而已。 

~~~java
@RequestMapping("/put")
public void put() {
    Book book = new Book();
    book.setName("红楼梦");
    restTemplate.put("http://HELLO-SERVICE/getbook3/{1}", book, 99);
}
~~~

### 4.4 DELETE请求

~~~java
@RequestMapping("/delete")
public void delete() {
    restTemplate.delete("http://HELLO-SERVICE/getbook4/{1}", 100);
}
~~~

delete方法也有几个重载的方法，不过重载的参数和前面基本一致，不赘述。 

## 5，断路器

### 5.1服务降级

​	当一个系统划分的模块越多时，如果一个模块出现故障会导致依赖它的模块也发生故障从而发生故障蔓延，进而导致整个服务的瘫痪，对于这个问题，Spring Cloud中最重要的解决方案就是断路器 。

1. 在服务消费者中引入依赖

   ~~~java
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-hystrix</artifactId>
   </dependency>
   ~~~

2. 修改服务消费者启动入口类

   ~~~java
   @EnableCircuitBreaker  //开启断路器
   @SpringBootApplication
   @EnableDiscoveryClient
   public class RibbonConsumerApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(RibbonConsumerApplication.class, args);
       }
       @LoadBalanced
       @Bean
       RestTemplate restTemplate() {
           return new RestTemplate();
       }
   }
   ~~~

3. 在具体调用类里面进行断路控制

   ~~~java
   @Service
   public class HelloService {
       @Autowired
       private RestTemplate restTemplate;
   
       @HystrixCommand(fallbackMethod = "error")
       public String hello() {
           ResponseEntity<String> responseEntity = restTemplate.getForEntity("http://HELLO-SERVICE/hello", String.class);
           return responseEntity.getBody();
       }
   
       public String error() {
           return "error";
       }
   }
   ~~~


+ RestTemplate执行网络请求的操作我们放在Service中来完成。 
+ error方法是一个请求失败时回调的方法。 
+ 在hello方法上通过@HystrixCommand注解来指定请求失败时回调的方法。 

不仅仅是服务提供者被关闭时我们需要断路器，如果请求超时也会触发熔断请求，调用回调方法返回数据 

### 5.2异常处理

​	我们在调用服务提供者时有可能会抛异常，默认情况下方法抛了异常会自动进行服务降级，交给服务降级中的方法去处理。

~~~java
@HystrixCommand(fallbackMethod = "error1")
public Book test2() {
    int i = 1 / 0;
    return restTemplate.getForObject("http://HELLO-SERVICE/getbook1", Book.class);
}

public Book error1(Throwable throwable) {
    System.out.println(throwable.getMessage());
    return new Book("百年孤独", 33, "马尔克斯", "人民文学出版社");
}
~~~

​	如果有一个异常抛出后我不希望进入到服务降级方法中去处理，而是直接将异常抛给用户，那么我们可以在@HystrixCommand注解中添加忽略异常。

~~~java

~~~

 ### 5.3请求缓存

​	缓存相关的注解一共有三个，分别是@CacheResult、@CacheKey和@CacheRemove。

**@CacheResult**

​	@CacheResult注解可以用在我们之前的Service方法上，表示给该方法开启缓存，默认情况下方法的所有参数都将作为缓存的key 。

**@CacheKey**

~~~java
@CacheResult
@HystrixCommand
public Book test6(@CacheKey Integer id,String aa) {
    return restTemplate.getForObject("http://HELLO-SERVICE/getbook5/{1}", Book.class, id);
}
~~~

​	这里我们使用@CacheKey注解指明了缓存的key为id，和aa这个参数无关，此时只要id相同就认为是同一个请求，而aa参数的值则不会作为判断缓存的依据（这里只是举例子，实际开发中我们的调用条件可能都要作为key，否则可能会获取到错误的数据）。如果我们即使用了@CacheResult中cacheKeyMethod属性来指定key，又使用了@CacheKey注解来指定key，则后者失效。 

**@CacheRemove**

​	这个当然是用来让缓存失效的注解，用法也很简单，如下：

```
@CacheRemove(commandKey = "test6")
@HystrixCommand
public Book test7(@CacheKey Integer id) {
    return null;
}
```

​	注意这里必须指定commandKey，commandKey的值就为缓存的位置，配置了commandKey属性的值，Hystrix才能找到请求命令缓存的位置。举个简单的例子，如下：

```java
@RequestMapping("/test6")
public Book test6() {
    HystrixRequestContext.initializeContext();
    //第一次发起请求
    Book b1 = bookService.test6(2);
    //清除缓存
    bookService.test7(2);
    //缓存被清除，重新发起请求
    Book b2 = bookService.test6(2);
    //参数一致，使用缓存数据
    Book b3 = bookService.test6(2);
    return b1;
}
```

### 5.4请求合并

​	Hystrix中的请求合并，就是利用一个合并处理器，将对同一个服务发起的连续请求合并成一个请求进行处理(这些连续请求的时间窗默认为10ms)，在这个过程中涉及到的一个核心类就是HystrixCollapser 。	

​	多个请求被合并为一个请求进行一次性处理，可以有效节省网络带宽和线程池资源，但是，有优点必然也有缺点，设置请求合并之后，本来一个请求可能5ms就搞定了，但是现在必须再等10ms看看还有没有其他的请求一起的，这样一个请求的耗时就从5ms增加到15ms了，不过，如果我们要发起的命令本身就是一个高延迟的命令，那么这个时候就可以使用请求合并了，因为这个时候时间窗的时间消耗就显得微不足道了，另外高并发也是请求合并的一个非常重要的场景。 





