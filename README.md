# dubbo

## 代码实例

> dubbo-demo

## Main方法的启动

```java
public class App 
{
    public static void main(String[] args)
    {
        Main.main(args);
    }
}
```

####  Main.main

```java
    
public static final String CONTAINER_KEY = "dubbo.container";

public static final String SHUTDOWN_HOOK_KEY = "dubbo.shutdown.hook";

private static final Logger logger = LoggerFactory.getLogger(Main.class);

private static final ExtensionLoader<Container> loader = 		    ExtensionLoader.getExtensionLoader(Container.class);
    
    private static volatile boolean running = true;
public static void main(String[] args) {
        try {
            if (args == null || args.length == 0) {
                String config = ConfigUtils.getProperty(CONTAINER_KEY, loader.getDefaultExtensionName());
                args = Constants.COMMA_SPLIT_PATTERN.split(config);
            }
            
            final List<Container> containers = new ArrayList<Container>();
            
            //dubbo的扩展机制
            for (int i = 0; i < args.length; i ++) {
                containers.add(loader.getExtension(args[i]));
            }
            
            logger.info("Use container type(" + Arrays.toString(args) + ") to run dubbo serivce.");
            
            if ("true".equals(System.getProperty(SHUTDOWN_HOOK_KEY))) {
	            Runtime.getRuntime().addShutdownHook(new Thread() {
	                public void run() {
	                    for (Container container : containers) {
	                        try {
	                            container.stop();
	                            logger.info("Dubbo " + container.getClass().getSimpleName() + " stopped!");
	                        } catch (Throwable t) {
	                            logger.error(t.getMessage(), t);
	                        }
	                        synchronized (Main.class) {
	                            running = false;
	                            Main.class.notify();
	                        }
	                    }
	                }
	            });
            }
            
            //dubbo提供的容器
            for (Container container : containers) {
                container.start();
                logger.info("Dubbo " + container.getClass().getSimpleName() + " started!");
            }
            System.out.println(new SimpleDateFormat("[yyyy-MM-dd HH:mm:ss]").format(new Date()) + " Dubbo service server started!");
        } catch (RuntimeException e) {
            e.printStackTrace();
            logger.error(e.getMessage(), e);
            System.exit(1);
        }
        synchronized (Main.class) {
            while (running) {
                try {
                    Main.class.wait();
                } catch (Throwable e) {
                }
            }
        }
    }
    
```

##### dubbo提供的容器

![](image/001.png)

##### SpringContainer

```java
    
public static final String DEFAULT_SPRING_CONFIG = "classpath*:META-INF/spring/*.xml";

public void start() {
        String configPath = ConfigUtils.getProperty(SPRING_CONFIG);
        if (configPath == null || configPath.length() == 0) {
            //默认加载META-INF/spring/*.xml
            configPath = DEFAULT_SPRING_CONFIG;
        }
      //加载spring
        context = new ClassPathXmlApplicationContext(configPath.split("[,\\s]+"));
        context.start();
    }

```



## 日志的集成

```java
public class LoggerFactory {
    	static {
	    String logger = System.getProperty("dubbo.application.logger");
	    if ("slf4j".equals(logger)) {
    		setLoggerAdapter(new Slf4jLoggerAdapter());
    	} else if ("jcl".equals(logger)) {
    		setLoggerAdapter(new JclLoggerAdapter());
    	} else if ("log4j".equals(logger)) {
    		setLoggerAdapter(new Log4jLoggerAdapter());
    	} else if ("jdk".equals(logger)) {
    		setLoggerAdapter(new JdkLoggerAdapter());
    	} else {
    		try {
    			setLoggerAdapter(new Log4jLoggerAdapter());
            } catch (Throwable e1) {
                try {
                	setLoggerAdapter(new Slf4jLoggerAdapter());
                } catch (Throwable e2) {
                    try {
                    	setLoggerAdapter(new JclLoggerAdapter());
                    } catch (Throwable e3) {
                        setLoggerAdapter(new JdkLoggerAdapter());
                    }
                }
            }
    	}
	}
}
```



## ADMIN控制台安装

1. 下载dubbo的源码

   > https://github.com/apache/incubator-dubbo-admin/

3. 修改webapp/WEB-INF/dubbo.properties

dubbo.registry.address=zookeeper的集群地址



> 控制中心是用来做服务治理的，比如控制服务的权重、服务的路由

## SIMPLE监控中心

Monitor也是一个dubbo服务，所以也会有端口和url

 

修改/conf目录下dubbo.properties /order-provider.xml

dubbo.registry.address=zookeeper的集群地址

 

监控服务的调用次数、调用关系、响应事件



## 启动服务检查

如果提供方没有启动的时候，默认会去检测所依赖的服务是否正常提供服务

如果check为false，表示启动的时候不去检查。当服务出现循环依赖的时候，check设置成false

dubbo:reference  属性： check默认值是true、false

 dubbo:consumer  check=”false”没有服务提供者的时候，报错

dubbo:registry  check=false注册订阅失败报错

## 多协议支持

dubbo支持的协议： dubbo、RMI、**hessian**、webservice、http、Thrift

#### hessian为例

##### 引入jar包

```xml
<dependency>
    <groupId>com.caucho</groupId>
    <artifactId>hessian</artifactId>
    <version>4.0.38</version>
</dependency>
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>servlet-api</artifactId>
    <version>2.5</version>
</dependency>
<dependency>
    <groupId>org.mortbay.jetty</groupId>
    <artifactId>jetty</artifactId>
    <version>6.1.26</version>
</dependency>
```

##### 修改provider.xml

```xml
  <dubbo:protocol name="hessian" port="20880" server="jetty"/>
```

##### 指定service服务的协议版本号

```xml
<!--发布的接口-->
<dubbo:service interface="cn.zhangspace.IOrderService" ref="OrderService" protocol="hessian"/>
```

##### 消费端改造

```xml
 <dubbo:reference id="orderServices" interface="cn.zhangspace.IOrderService" protocol="hessian"/>
```

## 多注册中心支持

```xml
<dubbo:registry  id ="one" address="zookeeper://192.168.78.129:2181" timeout="10000"/>

<dubbo:registry id ="two" address="zookeeper://192.168.78.128:2181" timeout="10000"/>
```

```xml
    <dubbo:service interface="cn.zhangspace.IOrderService" ref="OrderService" protocol="hessian" register="one"/>
    <dubbo:service interface="cn.zhangspace.IOrderService" ref="OrderService" protocol="hessian" register="two"/>
```

## 多版本支持

```xml

<dubbo:service interface="cn.zhangspace.IOrderService" ref="OrderService" version="0.0.0" protocol="hessian" />
<dubbo:service interface="cn.zhangspace.IOrderService" ref="OrderService2" version="0.0.1" protocol="hessian" />

<bean id="OrderService" class="cn.zhangspace.OrderServiceImpl"/>
<bean id="OrderService2" class="cn.zhangspace.OrderServiceImpl2"/>
```

```xml
<dubbo:reference id="orderServices" interface="cn.zhangspace.IOrderService" protocol="hessian" version="0.0.1"/>

```

## 异步调用

> hessian协议，使用async异步回调会报错

```xml
    <dubbo:reference id="orderServices" interface="cn.zhangspace.IOrderService" protocol="hessian" version="0.0.1" async="true"/>

```

```java
services.doOrder(request);
Future<DoOrderResponse> response1 = RpcContext.getContext().getFuture();
DoOrderResponse response = response1.get();
```

