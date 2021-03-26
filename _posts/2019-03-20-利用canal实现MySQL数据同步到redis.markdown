---
layout: post
title: "利用canal实现MySQL数据同步到redis"
date: "2019-03-20 11:34:12 +0800"
category: 数据库
tags: redis canal mysql
---

# 什么是canal

canal是阿里巴巴的一款基于数据库增量日志解析提供增量数据订阅与消费的产品，目前主要支持MySQL。

## 工作原理

canal的原理非常的简单如下图所示：

![canal工作原理](/images/2019-03-20-canal工作原理.jpeg)

1. canal模拟mysql slave的交互协议，伪装自己为mysql slave，向mysql master发送dump协议
2. mysql master收到dump请求，开始推送binary log给slave(也就是canal)
3. canal解析binary log对象(原始为byte流)

## 架构

![canal架构](/images/2019-03-20-canal架构.jpeg)

说明：

- server代表一个canal运行实例，对应于一个jvm
- instance对应于一个数据队列 （1个server对应1..n个instance)

instance模块：

- eventParser (数据源接入，模拟slave协议和master进行交互，协议解析)
- eventSink (Parser和Store链接器，进行数据过滤，加工，分发的工作)
- eventStore (数据存储)
- metaManager (增量订阅&消费信息管理器)

这里我们不做太多的分析：可以在[这里](<https://github.com/alibaba/canal/wiki/Introduction#%E6%9E%B6%E6%9E%84>)找到更加详细的说明。

# 快速开始

接下来，我们步入正题，究竟如何利用canal实现MySQL数据同步到redis。

## 配置MySQL

首先，我们需要配置一下MySQL，使其能够适配canal。

因为我一般使用docker来运行，所以这里给出的是docker的运行指南，采用其他的方式大同小异，不再给出。

### 生成镜像

首先，我们新建一个`MySQL`目录，并在该目录创建两个文件：

my.cnf

```properties
[mysqld]
log-bin=mysql-bin
binlog-format=ROW
server_id=1
```

Dockerfile

```dockerfile
FROM mysql:5.7
COPY my.cnf /etc/my.cnf
```

并在该目录下运行以下指令：

```
docker build -t mysql4canal:5.7 .
```

说明：

通过以上的操作，我们生成了一个可以生成Binlog的MySQL服务器镜像。

### 创建canal用户

首先，我们创建该镜像的一个容器。

```bash
docker run -d --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root mysql4canal:5.7
```

接下来，我们进入该容器。

```bash
docker exec -it mysql bash
```

进入该容器后，我们使用root用户登录mysql

```bash
mysql -uroot -proot
```

这样，我们就可以创建一个canal用户了

```mysql
CREATE USER canal IDENTIFIED BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
-- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
FLUSH PRIVILEGES;
```

至此，MySQL的配置已经结束了

## 启动canal

```bash
# 下载脚本
wget https://raw.githubusercontent.com/alibaba/canal/master/docker/run.sh 
chmod a+x run.sh

# 构建一个队列
./run.sh -e canal.auto.scan=false \
         -e canal.destinations=test \
         -e canal.instance.master.address=127.0.0.1:3306  \
         -e canal.instance.dbUsername=canal  \
         -e canal.instance.dbPassword=canal  \
         -e canal.instance.connectionCharset=UTF-8 \
         -e canal.instance.tsdb.enable=true \
         -e canal.instance.gtidon=false
```

至此，我们的canal服务端已经准备完毕了。

## redis根据canal返回日志更新数据

GitHub源码地址：[https://github.com/litong/canal2redis](https://github.com/litong/canal2redis)

### 1. 创建一个基于Maven的Spring Boot项目。

其依赖如下：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.14.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>

<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>1.8</java.version>
    <skipTests>true</skipTests>
    <canal.version>1.1.2</canal.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>com.alibaba.otter</groupId>
        <artifactId>canal.protocol</artifactId>
        <version>${canal.version}</version>
    </dependency>
    <dependency>
        <groupId>com.alibaba.otter</groupId>
        <artifactId>canal.client</artifactId>
        <version>${canal.version}</version>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

### 2. 创建Redis工具类

```java
/**
 * Redis工具类
 *
 * @author litong
 */
@Component
public class RedisUtil {

    /**
     * 默认过期时长（秒）
     */
    private final static long DEFAULT_EXPIRE = 1800;

    private final RedisTemplate<String, Object> redisTemplate;

    @Autowired
    public RedisUtil(RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    /**
     * 设置过期时长
     *
     * @param key     键
     * @param seconds 过期时长（秒）
     */
    public void expire(String key, long seconds) {
        if (seconds > 0) {
            redisTemplate.expire(key, seconds, TimeUnit.SECONDS);
        }
    }

    /**
     * 设置默认的过期时长
     *
     * @param key 键
     */
    public void expireDefault(String key) {
        redisTemplate.expire(key, DEFAULT_EXPIRE, TimeUnit.SECONDS);
    }

    /**
     * 获取过期时长（秒）
     *
     * @param key 键
     * @return 时长（秒）
     */
    public long getExpire(String key) {
        return redisTemplate.getExpire(key, TimeUnit.SECONDS);
    }

    /**
     * 是否包含某键
     *
     * @param key 键
     * @return true 包含 false 不包含
     */
    public boolean hasKey(String key) {
        return redisTemplate.hasKey(key);
    }

    /**
     * 删除某键
     *
     * @param key 键
     */
    public void delete(String... key) {
        if (key != null && key.length > 0) {
            if (key.length == 1) {
                redisTemplate.delete(key[0]);
            } else {
                redisTemplate.delete(CollectionUtils.arrayToList(key));
            }
        }
    }

    /**
     * 获取key的value
     *
     * @param key 键
     * @return 值
     */
    public Object get(String key) {
        return key == null ? null : redisTemplate.opsForValue().get(key);
    }

    /**
     * 永久set
     *
     * @param key   键
     * @param value 值
     */
    public void set(String key, Object value) {
        redisTemplate.opsForValue().set(key, value);
    }

    /**
     * 默认set 默认实效时间 DEFAULT_EXPIRE
     *
     * @param key   键
     * @param value 值
     */
    public void setDefault(String key, Object value) {
        this.set(key, value, DEFAULT_EXPIRE);
    }

    /**
     * 普通缓存放入并设置时间
     *
     * @param key     键
     * @param value   值
     * @param seconds 时间(秒)
     */
    public void set(String key, Object value, long seconds) {
        if (seconds > 0) {
            redisTemplate.opsForValue().set(key, value, seconds, TimeUnit.SECONDS);
        } else {
            set(key, value);
        }
    }

    /**
     * 递增
     *
     * @param key   键
     * @param delta 步长
     * @return 递增之后的结果
     */
    public long incr(String key, long delta) {
        if (delta <= 0) {
            throw new RuntimeException("步长必须大于0");
        }
        return redisTemplate.opsForValue().increment(key, delta);
    }

    /**
     * 递减
     *
     * @param key   键
     * @param delta 步长
     * @return 递减之后的结果
     */
    public long decr(String key, long delta) {
        if (delta <= 0) {
            throw new RuntimeException("步长必须大于0");
        }
        return redisTemplate.opsForValue().increment(key, -delta);
    }
}
```

该工具类中，对常见的Redis读写等操作做了一层封装。

### 3. 创建Redis的配置类

```java
/**
 * Redis配置类
 * @author litong
 */
@Configuration
public class RedisConfig {
    private final RedisConnectionFactory factory;

    @Autowired
    public RedisConfig(RedisConnectionFactory factory) {
        this.factory = factory;
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashValueSerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new StringRedisSerializer());
        redisTemplate.setConnectionFactory(factory);
        return redisTemplate;
    }
}
```

### 4. 创建canal的配置类

```java
/**
 * Canal配置类
 * @author litong
 */
@Slf4j
@Setter
@Component
@ConfigurationProperties("canal")
public class CanalConfig {

    private String host;
    private Integer port;
    private String destination;
    private String username;
    private String password;
    private String subscribe;

    @Bean
    public CanalConnector getCanalConnector() {
        CanalConnector canalConnector = CanalConnectors.newClusterConnector(Lists.newArrayList(new InetSocketAddress(host, port)), destination, username, password);
        canalConnector.connect();
        // 指定要订阅的数据库和表
        canalConnector.subscribe(subscribe);
        // 回滚到上次中断的位置
        canalConnector.rollback();
        log.info("canal客户端启动......");
        return canalConnector;
    }
}
```

该类能够在SpringBoot启动的时候创建Canal的客户端，这也是整个项目的基础。

### 5. 数据操作类

当数据库发生变动的时候，将其同步至redis，这些都是由数据操作类进行。

#### 操作基类

数据的操作无非是增删改查，而这些都是有共通的地方，所以，将其抽象为数据操作基类，供各个处理类继承。

```java
/**
 * 数据库操作处理基类
 *
 * @author litong
 */
@Slf4j
public abstract class AbstractHandler {

    /**
     * Redis工具实例
     */
    @Autowired
    protected RedisUtil redisUtil;

    /**
     * 下一个执行者
     */
    AbstractHandler nextHandler;

    /**
     * 事件类型
     */
    EventType eventType;

    /**
     * 处理Canal消息
     *
     * @param entry Canal消息
     */
    public void handleMessage(Entry entry) {
        if (this.eventType == entry.getHeader().getEventType()) {
            //发生写入操作的库名
            String database = entry.getHeader().getSchemaName();
            //发生写入操作的表名
            String table = entry.getHeader().getTableName();
            log.info("监听到数据库：{}，表：{} 的 {} 事件", database, table, eventType.toString());

            Optional.ofNullable(this.getRowChange(entry))
                    .ifPresent(rowChange -> handleRowChange(database, table, rowChange));
        } else {
            if (nextHandler != null) {
                nextHandler.handleMessage(entry);
            }
        }
    }

    /**
     * 处理数据库数据
     *
     * @param database  数据库名称
     * @param table     表名称
     * @param rowChange 行更改数据
     */
    public abstract void handleRowChange(String database, String table, RowChange rowChange);

    /**
     * 获得发生操作的数据
     *
     * @param entry Canal消息
     * @return 行更改数据
     */
    private RowChange getRowChange(Entry entry) {
        RowChange rowChange = null;
        try {
            rowChange = RowChange.parseFrom(entry.getStoreValue());
        } catch (InvalidProtocolBufferException e) {
            log.error("根据CanalEntry获取RowChange异常:", e);
        }
        return rowChange;
    }

    /**
     * 行数据转Map数据
     */
    Map<String, String> columnsToMap(List<Column> columns) {
        return columns.stream().collect(Collectors.toMap(Column::getName, Column::getValue));
    }
}
```

增删改查的代码不再给出，可在以上给出的源码地址中查阅。

### 6. 线程池配置

为了保证执行的效率，采用了多线程的方式进行解析。

```java
/**
 * 线程池配置类
 *
 * @author litong
 */
@EnableAsync
@Configuration
public class ThreadPoolConfig {

    @Bean("canal")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(200);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("canal-");
        // 重试添加当前的任务
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return executor;
    }
}
```

### 7. 创建canal定时任务

最后，也是最重要的一点，创建cannal定时任务。程序通过不断的定时任务，拉取MySQL的Binlog，并通过binlog的内容尽心不同的redis操作，从而实现redis与MySQL的同步。

```java
/**
 * Canal定时任务
 *
 * @author litong
 */
@Slf4j
@Component
public class CanalSchedule {

    private final CanalConnector canalConnector;

    private final InsertHandler insertHandler;

    @Value("${canal.batchSize}")
    private int batchSize;

    @Autowired
    public CanalSchedule(CanalConnector canalConnector, InsertHandler insertHandler) {
        this.canalConnector = canalConnector;
        this.insertHandler = insertHandler;
    }

    @Async("canal")
    @Scheduled(fixedDelay = 200)
    public void fetch() {
        try {
            Message message = canalConnector.getWithoutAck(batchSize);
            long batchId = message.getId();
            log.debug("batchId={}", batchId);
            try {
                List<Entry> entries = message.getEntries();
                if (batchId != -1 && entries.size() > 0) {
                    entries.forEach(entry -> {
                        if (entry.getEntryType() == EntryType.ROWDATA) {
                            insertHandler.handleMessage(entry);
                        }
                    });
                }
                canalConnector.ack(batchId);
            } catch (Exception e) {
                log.error("批量获取 mysql 同步信息失败，batchId回滚,batchId=" + batchId, e);
                canalConnector.rollback(batchId);
            }
        } catch (Exception e) {
            log.error("Canal定时任务异常！", e);
        }
    }
}
```

至此，利用canal实现MySQL数据同步到redis的实现也就完成了。