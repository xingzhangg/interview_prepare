

## [Redis两种方式实现限流](https://www.cnblogs.com/barrywxx/p/8563533.html)

# IP限流

实现访问频率限制: 实现访问者 $ip （UID） 在一定的时间 $time 内只能访问 $limit 次.

### 非脚本实现

```java
private boolean accessLimit(String ip, int limit, int time, Jedis jedis) {
    boolean result = true;

    String key = "rate.limit:" + ip;
    if (jedis.exists(key)) {
        long afterValue = jedis.incr(key);
        if (afterValue > limit) {
            result = false;
        }
    } else {
        Transaction transaction = jedis.multi();
        transaction.incr(key);
        transaction.expire(key, time);
        transaction.exec();
    }
    return result;
}
```

以上代码有两点缺陷 

1. 可能会出现竞态条件: 解决方法是用 `WATCH` 监控 `rate.limit:$IP` 的变动, 但较为麻烦;
2. 以上代码在不使用 `pipeline` 的情况下最多需要向Redis请求5条指令, 传输过多.

### Lua脚本实现 

Redis 允许将 Lua 脚本传到 Redis 服务器中执行, 脚本内可以调用大部分 Redis 命令, 且 Redis 保证脚本的**原子性**:

- 首先需要准备Lua代码: script.lua

```lua
--
-- Created by IntelliJ IDEA.
-- User: jifang
-- Date: 16/8/24
-- Time: 下午6:11
--

local key = "rate.limit:" .. KEYS[1]
local limit = tonumber(ARGV[1])
local expire_time = ARGV[2]

local is_exists = redis.call("EXISTS", key)
if is_exists == 1 then
    if redis.call("INCR", key) > limit then
        return 0
    else
        return 1
    end
else
    redis.call("SET", key, 1)
    redis.call("EXPIRE", key, expire_time)
    return 1
end
```

- Java

``` JAVA
  private boolean accessLimit(String ip, int limit, int timeout, Jedis connection) throws IOException {
      List<String> keys = Collections.singletonList(ip);
      List<String> argv = Arrays.asList(String.valueOf(limit), String.valueOf(timeout));
  
      return 1 == (long) connection.eval(loadScriptString("script.lua"), keys, argv);
  }
  
  // 加载Lua代码
  private String loadScriptString(String fileName) throws IOException {
      Reader reader = new InputStreamReader(Client.class.getClassLoader().getResourceAsStream(fileName));
      return CharStreams.toString(reader);
  }
```

Lua 嵌入 Redis 优势: 

1. 减少网络开销: 不使用 Lua 的代码需要向 Redis 发送多次请求, 而脚本只需一次即可, 减少网络传输;
2. 原子操作: Redis 将整个脚本作为一个原子执行, 无需担心并发, 也就无需事务;
3. 复用: 脚本会永久保存 Redis 中, 其他客户端可继续使用.



# 时间限流

首先需要准备Lua代码: script.lua

```lua
local key = KEYS[1] --限流KEY（一秒一个）
local limit = tonumber(ARGV[1]) --限流大小
local current = tonumber(redis.call('get', key) or "0")
if current + 1 > limit then --如果超出限流大小
    return 0
else --请求数+1，并设置2秒过期
    redis.call("INCRBY", key,"1")
    redis.call("expire", key,"2")
end
return 1
```

```java
import org.apache.commons.io.FileUtils;

import redis.clients.jedis.Jedis;

import java.io.File;
import java.io.IOException;
import java.net.URISyntaxException;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;

public class RedisLimitRateWithLUA {

    public static void main(String[] args) {
        final CountDownLatch latch = new CountDownLatch(1);

        for (int i = 0; i < 7; i++) {
            new Thread(new Runnable() {
                public void run() {
                    try {
                        latch.await();
                        System.out.println("请求是否被执行："+accquire());
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }).start();

        }

        latch.countDown();
    }

    public static boolean accquire() throws IOException, URISyntaxException {
        Jedis jedis = new Jedis("127.0.0.1");
        File luaFile = new File(RedisLimitRateWithLUA.class.getResource("/").toURI().getPath() + "limit.lua");
        String luaScript = FileUtils.readFileToString(luaFile);

        String key = "ip:" + System.currentTimeMillis()/1000; // 当前秒
        String limit = "5"; // 最大限制
        List<String> keys = new ArrayList<String>();
        keys.add(key);
        List<String> args = new ArrayList<String>();
        args.add(limit);
        Long result = (Long)(jedis.eval(luaScript, keys, args)); // 执行lua脚本，传入参数
        return result == 1;
    }
}
```

- 每次请求时将当前时间(精确到秒)作为 Key 写入到 Redis 中，超时时间设置为 2 秒，Redis 将该 Key 的值进行自增。
- 当达到阈值时返回错误。
- 写入 Redis 的操作用 Lua 脚本来完成，利用 Redis 的单线程机制可以保证每个 Redis 请求的原子性。