# Redis集群与redisMq
## 现象
利用redis实现简易消息队列。本地开发，消息正常消费；服务端，消息始终未消费，且未发现异常信息

## 背景及原因
利用Redis实现的带ACK的Mq，如下：
```java
public class RedisMQ {
    private static final String PROCESS_LIST_SUFFIX = ":PROCESSING";

    private JedisPool jedisPool;
    private String topic;
    private String topicProcessing;

    public RedisMQ(JedisPool jedisPool, String topic) {
        this.jedisPool = jedisPool;
        this.topic = topic;
        topicProcessing = topic + PROCESS_LIST_SUFFIX;
    }

    public void produce(String message){
        try (Jedis jedis = jedisPool.getResource()) {
            jedis.lpush(topic, message);
        }
    }

    public String consume() {
        try (Jedis jedis = jedisPool.getResource()) {
            return jedis.rpoplpush(topic, topicProcessing);
        }
    }

    public String consumeAndAck() {
        try (Jedis jedis = jedisPool.getResource()) {
            return jedis.rpop(topic);
        }
    }

    public void ack(String message) {
        try (Jedis jedis = jedisPool.getResource()) {
            jedis.lrem(topicProcessing, 0, message);
        }
    }
}
```

这里调用consume方法消费信息，消费后调用ack方法确认消息已成功消费。结果在服务端使用发现消息处于topic队列中，不存在于topicProcessing队列。消费过程在ScheduledExecutorService中执行，大致如下：
```
@Override
public void run() {
    String message = redisMQ.consume();
    handle(message);
    redisMQ.ack(message);
}
```
由于这个run()方法中执行抛的异常并未打印出来，因此服务端未看到任何异常信息。
使用try-catch包裹run方法全部代码后发现如下异常：
```
redis.clients.jedis.exceptions.JedisDataException: ERR 'RPOPLPUSH' command keys must in same slot
```
因此这里也找到了原因： 服务器用的redis-cluster（阿里云的redis），而在集群模式下RPOPLPUSH命令的key必须在同一个slot下。

## 思考
### Spring Boot会吃掉线程池中的异常信息吗？
简单测试
```java
public class ThreadPoolTest {
    private static final Logger logger = LoggerFactory.getLogger(ThreadPoolTest.class);

    private static AtomicInteger count = new AtomicInteger(0);
    private static class Task implements Runnable {
        @Override
        public void run() {
            logger.info("invoke Task.run()");
            count.incrementAndGet();
            int rem = count.intValue() % 3;
            logger.info(String.valueOf(100 / rem));
        }
    }

    public static void main(String[] args) {
        ScheduledExecutorService ses = Executors.newSingleThreadScheduledExecutor();
        ses.scheduleWithFixedDelay(new Task(), 1000, 1000, TimeUnit.MILLISECONDS);
    }
}
```
运行后结果如下：
```
17:52:58.116 [pool-1-thread-1] INFO com.fifedu.kyxl.kse.ThreadPoolTest - invoke Task.run()
17:52:58.140 [pool-1-thread-1] INFO com.fifedu.kyxl.kse.ThreadPoolTest - 100
17:52:59.143 [pool-1-thread-1] INFO com.fifedu.kyxl.kse.ThreadPoolTest - invoke Task.run()
17:52:59.143 [pool-1-thread-1] INFO com.fifedu.kyxl.kse.ThreadPoolTest - 50
17:53:00.144 [pool-1-thread-1] INFO com.fifedu.kyxl.kse.ThreadPoolTest - invoke Task.run()
```
可以看出这里的RuntimeException并没有打印出来，因此**不仅是Spring Boot，线程池都会吃掉RuntimeException异常信息**

同时也可以看出 `Executors.newSingleThreadScheduledExecutor` 在遇到 `RuntimeException` 后唯一的线程就“死”了，自然以后的任务都不会执行了（但是这时主线程并未结束）。
> 换成 `Executors.newScheduledThreadPool(1)` 也是同样

改成如下（对异常进行捕获）后，则Task就可以一直执行下去了：
```java
public class ThreadPoolTest {
    private static final Logger logger = LoggerFactory.getLogger(ThreadPoolTest.class);

    private static AtomicInteger count = new AtomicInteger(0);
    private static class Task implements Runnable {
        @Override
        public void run() {
            logger.info("invoke Task.run()");
            count.incrementAndGet();
            int rem = count.intValue() % 3;
            try {
                logger.info(String.valueOf(100 / rem));
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        ScheduledExecutorService ses = Executors.newSingleThreadScheduledExecutor();
        ses.scheduleWithFixedDelay(new Task(), 1000, 1000, TimeUnit.MILLISECONDS);
    }
}
```

### 线程方法体需要置于try-catch中吗？
有必要
```
@Override
public void run() {
    try {
        doTask();
    } catch (Exception e) {
        logger.error("Xxx task run failed.",e);
    }
}

private void doTask() {
    String message = redisMQ.consume();
    handle(message);
    redisMQ.ack(message);
}
```

### 以上情景如何改进
使用redis cluster的hash-tag使topic与topicProcessing定位到同一个slot。参见[官方文档](https://redis.io/topics/cluster-spec#keys-hash-tags)
```java
public class RedisMQ {
    private static final String PROCESS_LIST_SUFFIX = ":PROCESSING";
    public static final String HASH_TAG_PREFIX = "{";
    public static final String HASH_TAG_SUFFIX = "}";

    private JedisPool jedisPool;
    private String hashTagTopic;
    private String hashTagTopicProcessing;

    public RedisMQ(JedisPool jedisPool, String rawTopic) {
        this.jedisPool = jedisPool;
        this.hashTagTopic = HASH_TAG_PREFIX + rawTopic + HASH_TAG_SUFFIX;
        hashTagTopicProcessing = this.hashTagTopic + PROCESS_LIST_SUFFIX;
    }

    public void produce(String message){
        try (Jedis jedis = jedisPool.getResource()) {
            jedis.lpush(hashTagTopic, message);
        }
    }

    public String consume() {
        try (Jedis jedis = jedisPool.getResource()) {
            return jedis.rpoplpush(hashTagTopic, hashTagTopicProcessing);
        }
    }

    public String consumeAndAck() {
        try (Jedis jedis = jedisPool.getResource()) {
            return jedis.rpop(hashTagTopic);
        }
    }

    public void ack(String message) {
        try (Jedis jedis = jedisPool.getResource()) {
            jedis.lrem(hashTagTopicProcessing, 0, message);
        }
    }
}
```

