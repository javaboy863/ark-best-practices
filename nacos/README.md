# 1、nacos应该怎么用？
### a、当配置中心使用
配置中心接入：1.nacos配置中心接入

### b、当注册中心使用
注册中心接入：2.nacos注册中心——生产环境改造配置

### C、使用nacos读取配置的正确使用姿势
新项目使用方法：1.nacos配置中心接入
老项目使用方法：d、像使用redis一样，每次从nacos读取data-id的问题

# 2、nacos有几个环境？
nacos环境信息：2.nacos配置中心使用和管理规范


# 3、我们再使用nacos过程中有哪些问题？
### a、注入的配置项不能自动刷新问题
- 当使用@Value注入nacos管理的配置项后，必须在注入的类上加入@RefreshScope，然后在nacos修改配置后，注入的配置项才能动态刷新。
```java
@RefreshScope
public class DoubleWriteConfig {
    @Value("${grayscale.platform.list:[]}")
    private String grayscalePlatform;

    @Value("${grayscale.double.type:0}")
    private Integer doubleType;
}
```

### b、一个项目中使用多个nacos管理的配置文件问题
- 使用场景：按多个配置文件归类的场景，比如redis配置信息单独存放一个配置文件
```properties
1、bootstrap.properties新增配置：

spring.cloud.nacos.config.ext-config[0].data-id=redis.properties
spring.cloud.nacos.config.ext-config[0].group=DEFAULT_GROUP
spring.cloud.nacos.config.ext-config[0].refresh=true

spring.cloud.nacos.config.ext-config[1].data-id=datasource.properties
spring.cloud.nacos.config.ext-config[1].group=DEFAULT_GROUP
spring.cloud.nacos.config.ext-config[1].refresh=true
2、正常使用@Value注解注入配置项即可。
```

### c、配置项（data-id）使用太多太零散的问题
问题：如下图，独立的配置项太多，每个配置项都用独立一个data-id来配置会造成如下问题：
- 配置项太多太杂，时间长了，不知道哪些再用哪些已废弃
- 不知道这些配置，关联的有几个项目再用
- 配置项太过于孤立，无法提现相互之间的关系
- 用法错误，一个data-id应该是一组配置体的集合


解决方法：
- 按配置文件方式管理配置，每个项目对应一个配置文件，所有项目的配置信息都在该配置文件中管理。参考：1.nacos配置中心接入
- 对于多个配置文件的场景，参考：“b、一个项目中使用多个nacos管理的配置文件问题”
- 新项目，原则上不允许以硬编码的方式单独读取单个data-id

### d、像使用redis一样，每次从nacos读取data-id的问题
- 问题：由于历史遗留问题，通过硬编码的方式读取data-id，每执行一次就会从nacos服务器中读取一次配置，会造成服务自身的性能问题和nacos服务器负载过高。注：不推荐通过nacos api单个使用和读取data-id。
- 解决方法： 示例代码：wd-distribution nacos_data_id_use_demo分支
```java
1、POM增加依赖
<dependency>
   <groupId>com.alibaba.nacos</groupId>
   <artifactId>nacos-spring-context</artifactId>
   <version>1.1.1</version>
</dependency>
<dependency>
   <groupId>com.alibaba.spring</groupId>
   <artifactId>spring-context-support</artifactId>
   <version>1.0.11</version>
</dependency>

2、读取nacos配置的key
@Data
@Component
@EnableNacosConfig(globalProperties = @NacosProperties(serverAddr = "${spring.cloud.nacos.config.server-addr}"))
public class NacosConfigKey implements InitializingBean {

    public String redisHost;

    @NacosInjected
    private ConfigService configService;

    @NacosConfigListener(dataId = "redisHost")
    public void onMessage(String redisHost) {
        this.redisHost = redisHost;
    }

    private String getConfig(String config) {
        try {
            String value = configService.getConfig(config, "DEFAULT_GROUP", 3000L);
            return value;
        } catch (NacosException e) {
            e.printStackTrace();
        }
        return null;
    }


    @Override
    public void afterPropertiesSet() throws Exception {
        this.redisHost = getConfig("redisHost");
    }
}


3、使用nacos配置的key
@Resource
private NacosConfigKey nacosConfigKey;
System.out.println(nacosConfigKey.getRedisHost());

```

### e、使用注解加载多个data-id
```java
1、通过注解加载需要的data-id
@NacosPropertySources({
@NacosPropertySource(dataId = "redis.properties", autoRefreshed = true)
})


2、通过@NacosValue注解注入上面定义的data-id的key（必须是key，value形式）
@NacosValue(value = "${redis.host:1}", autoRefreshed = true)
private String redis;
```