## 普通实现

既然是选用了 Redis，那么它就得具有排他性才行。同时它最好也有锁的一些基本特性：

- 高性能(加、解锁时高性能)
- 可以使用阻塞锁与非阻塞锁。
- 不能出现死锁。
- 可用性(不能出现节点 down 掉后加锁失败)。

这里利用 `Redis set key` 时的一个 NX 参数可以保证在这个 key 不存在的情况下写入成功。并且再加上 EX 参数可以让该 key 在超时之后自动删除。

所以利用以上两个特性可以保证在同一时刻只会有一个进程获得锁，并且不会出现死锁(最坏的情况就是超时自动删除 key)。


### 加锁

实现代码如下：

```java

    private static final String SET_IF_NOT_EXIST = "NX";
    private static final String SET_WITH_EXPIRE_TIME = "PX";
    
    public  boolean tryLock(String key, String request) {
        String result = this.jedis.set(LOCK_PREFIX + key, request, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, 10 * TIME);

        if (LOCK_MSG.equals(result)){
            return true ;
        }else {
            return false ;
        }
    }
```

注意这里使用的 jedis 的api。

```java
String set(String key, String value, String nxxx, String expx, long time);
```

**该命令可以保证 NX EX 的原子性。一定不要把两个命令(NX EX)分开执行，如果在 NX 之后程序出现问题就有可能产生死锁。**

#### 阻塞锁
同时也可以实现一个阻塞锁：

```java
    //一直阻塞
    public void lock(String key, String request) throws InterruptedException {

        for (;;){
            String result = this.jedis.set(LOCK_PREFIX + key, request, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, 10 * TIME);
            if (LOCK_MSG.equals(result)){
                break ;
            }
				
			  //防止一直消耗 CPU 	
            Thread.sleep(DEFAULT_SLEEP_TIME) ;
        }

    }
    
     //自定义阻塞时间
     public boolean lock(String key, String request,int blockTime) throws InterruptedException {

        while (blockTime >= 0){

            String result = this.jedis.set(LOCK_PREFIX + key, request, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, 10 * TIME);
            if (LOCK_MSG.equals(result)){
                return true ;
            }
            blockTime -= DEFAULT_SLEEP_TIME ;

            Thread.sleep(DEFAULT_SLEEP_TIME) ;
        }
        return false ;
    }

```

### 解锁

解锁也很简单，其实就是把这个 key 删掉就万事大吉了，比如使用 `del key` 命令。

但现实往往没有那么 easy。

如果进程 A 获取了锁设置了超时时间，但是由于执行周期较长导致到了超时时间之后锁就自动释放了。这时进程 B 获取了该锁执行很快就释放锁。这样就会出现进程 B 将进程 A 的锁释放了。

所以最好的方式是在每次解锁时都需要判断锁**是否是自己**的。

这时就需要结合加锁机制一起实现了。

加锁时需要传递一个参数，将该参数作为这个 key 的 value，这样每次解锁时判断 value 是否相等即可。

所以解锁代码就不能是简单的 `del`了。

```java
    public  boolean unlock(String key,String request){
        //lua script
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";

        Object result = null ;
        if (jedis instanceof Jedis){
            result = ((Jedis)this.jedis).eval(script, Collections.singletonList(LOCK_PREFIX + key), Collections.singletonList(request));
        }else if (jedis instanceof JedisCluster){
            result = ((JedisCluster)this.jedis).eval(script, Collections.singletonList(LOCK_PREFIX + key), Collections.singletonList(request));
        }else {
            //throw new RuntimeException("instance is error") ;
            return false ;
        }

        if (UNLOCK_MSG.equals(result)){
            return true ;
        }else {
            return false ;
        }
    }
```

这里使用了一个 `lua` 脚本来判断 value 是否相等，相等才执行 del 命令。

使用 `lua` 也可以保证这里两个操作的原子性。

因此上文提到的四个基本特性也能满足了：

- 使用 Redis 可以保证性能。
- 阻塞锁与非阻塞锁见上文。
- 利用超时机制解决了死锁。
- Redis 支持集群部署提高了可用性。



## 基于lua脚本执行

说道Redis分布式锁大部分人都会想到： `setnx+lua`，或者知道 `set key value px milliseconds nx`。后一种方式的核心实现命令如下：

```
- 获取锁（unique_value可以是UUID等）

SET resource_name unique_value NX PX 30000


- 释放锁（lua脚本中，一定要比较value，防止误解锁）

if redis.call("get",KEYS[1]) == ARGV[1] then

 return redis.call("del",KEYS[1])

else

 return 0

end
```



这种实现方式有3大要点（也是面试概率非常高的地方）：

1. set命令要用 `set key value px milliseconds nx`；
2. value要具有唯一性；
3. 释放锁时要验证value值，不能误解锁；

事实上这类琐最大的缺点就是它加锁时只作用在一个Redis节点上，即使Redis通过sentinel保证高可用，如果这个master节点由于某些原因发生了主从切换，那么就会出现锁丢失的情况：

1. 在Redis的master节点上拿到了锁；
2. 但是这个加锁的key还没有同步到slave节点；
3. master故障，发生故障转移，slave节点升级为master节点；
4. 导致锁丢失。

正因为如此，Redis作者antirez基于分布式环境下提出了一种更高级的分布式锁的实现方式：**Redlock**。笔者认为，Redlock也是Redis所有分布式锁实现方式中唯一能让面试官高潮的方式。



## Redlock实现

在Redis的分布式环境中，我们假设有N个Redis master。这些节点**完全互相独立，不存在主从复制或者其他集群协调机制**。我们确保将在N个实例上使用与在Redis单实例下相同方法获取和释放锁。现在我们假设有5个Redis master节点，同时我们需要在5台服务器上面运行这些Redis实例，这样保证他们不会同时都宕掉。

为了取到锁，客户端应该执行以下操作:

- 获取当前Unix时间，以毫秒为单位。
- 依次尝试从5个实例，使用相同的key和**具有唯一性的value**（例如UUID）获取锁。当向Redis请求获取锁时，客户端应该设置一个网络连接和响应超时时间，这个超时时间应该小于锁的失效时间。例如你的锁自动失效时间为10秒，则超时时间应该在5-50毫秒之间。这样可以避免服务器端Redis已经挂掉的情况下，客户端还在死死地等待响应结果。如果服务器端没有在规定时间内响应，客户端应该尽快尝试去另外一个Redis实例请求获取锁。
- 客户端使用当前时间减去开始获取锁时间（步骤1记录的时间）就得到获取锁使用的时间。**当且仅当从大多数**（N/2+1，这里是3个节点）**的Redis节点都取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功**。
- 如果取到了锁，key的真正有效时间等于有效时间减去获取锁所使用的时间（步骤3计算的结果）。
- 如果因为某些原因，获取锁失败（没有在至少N/2+1个Redis实例取到锁或者取锁时间已经超过了有效时间），客户端应该在**所有的Redis实例上进行解锁**（即便某些Redis实例根本就没有加锁成功，防止某些节点获取到锁但是客户端没有得到响应而导致接下来的一段时间不能被重新获取锁）。

实现方式：

* https://www.jianshu.com/p/f302aa345ca8
* https://www.jianshu.com/p/7e47a4503b87