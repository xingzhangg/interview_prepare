# 一致 Hash 算法

当我们在做数据库分库分表或者是分布式缓存时，不可避免的都会遇到一个问题:

如何将数据均匀的分散到各个节点中，并且尽量的在加减节点时能使受影响的数据最少。

## Hash 取模
随机放置就不说了，会带来很多问题。通常最容易想到的方案就是 `hash 取模`了。

可以将传入的 Key 按照 `index = hash(key) % N` 这样来计算出需要存放的节点。其中 hash 函数是一个将字符串转换为正整数的哈希映射方法，N 就是节点的数量。

这样可以满足数据的均匀分配，但是这个算法的容错性和扩展性都较差。

比如增加或删除了一个节点时，所有的 Key 都需要重新计算，显然这样成本较高，为此需要一个算法满足分布均匀同时也要有良好的容错性和拓展性。

## 一致 Hash 算法

一致 Hash 算法是将所有的哈希值构成了一个环，其范围在 `0 ~ 2^32-1`。如下图：

![](https://ws1.sinaimg.cn/large/006tNc79gy1fn8kbmd4ncj30ad08y3yn.jpg)

之后将各个节点散列到这个环上，可以用节点的 IP、hostname 这样的唯一性字段作为 Key 进行 `hash(key)`，散列之后如下：

![](https://ws3.sinaimg.cn/large/006tNc79gy1fn8kf72uwuj30a40a70t5.jpg)

之后需要将数据定位到对应的节点上，使用同样的 `hash 函数` 将 Key 也映射到这个环上。

![](https://ws3.sinaimg.cn/large/006tNc79gy1fn8kj9kd4oj30ax0aomxq.jpg)

这样按照顺时针方向就可以把 k1 定位到 `N1节点`，k2 定位到 `N3节点`，k3 定位到 `N2节点`。

### 容错性
这时假设 N1 宕机了：

![](https://ws3.sinaimg.cn/large/006tNc79gy1fn8kl9pp06j30a409waaj.jpg)

依然根据顺时针方向，k2 和 k3 保持不变，只有 k1 被重新映射到了 N3。这样就很好的保证了容错性，当一个节点宕机时只会影响到少少部分的数据。

### 拓展性

当新增一个节点时:

![](https://ws1.sinaimg.cn/large/006tNc79gy1fn8kp1fc9xj30ca0abt9c.jpg)

在 N2 和 N3 之间新增了一个节点 N4 ，这时会发现受印象的数据只有 k3，其余数据也是保持不变，所以这样也很好的保证了拓展性。

## 虚拟节点
到目前为止该算法依然也有点问题:

当节点较少时会出现数据分布不均匀的情况：

![](https://ws2.sinaimg.cn/large/006tNc79gy1fn8krttekbj30c10a5dg5.jpg)

这样会导致大部分数据都在 N1 节点，只有少量的数据在 N2 节点。

为了解决这个问题，一致哈希算法引入了虚拟节点。将每一个节点都进行多次 hash，生成多个节点放置在环上称为虚拟节点:

![](https://ws2.sinaimg.cn/large/006tNc79gy1fn8ktzuswkj30ae0abdgb.jpg)

计算时可以在 IP 后加上编号来生成哈希值。

这样只需要在原有的基础上多一步由虚拟节点映射到实际节点的步骤即可让少量节点也能满足均匀性。

### **一致性hash算法的Java实现**

#### **1、不带虚拟节点的**

``` java
package hash;
 
import java.util.SortedMap;
import java.util.TreeMap;
 
/**
 * 不带虚拟节点的一致性Hash算法
 */
public class ConsistentHashingWithoutVirtualNode {
 
	//待添加入Hash环的服务器列表
	private static String[] servers = { "192.168.0.0:111", "192.168.0.1:111",
			"192.168.0.2:111", "192.168.0.3:111", "192.168.0.4:111" };
 
	//key表示服务器的hash值，value表示服务器
	private static SortedMap<Integer, String> sortedMap = new TreeMap<Integer, String>();
 
	//程序初始化，将所有的服务器放入sortedMap中
	static {
		for (int i=0; i<servers.length; i++) {
			int hash = getHash(servers[i]);
			System.out.println("[" + servers[i] + "]加入集合中, 其Hash值为" + hash);
			sortedMap.put(hash, servers[i]);
		}
		System.out.println();
	}
 
	//得到应当路由到的结点
	private static String getServer(String key) {
		//得到该key的hash值
		int hash = getHash(key);
		//得到大于该Hash值的所有Map
		SortedMap<Integer, String> subMap = sortedMap.tailMap(hash);
		if(subMap.isEmpty()){
			//如果没有比该key的hash值大的，则从第一个node开始
			Integer i = sortedMap.firstKey();
			//返回对应的服务器
			return sortedMap.get(i);
		}else{
			//第一个Key就是顺时针过去离node最近的那个结点
			Integer i = subMap.firstKey();
			//返回对应的服务器
			return subMap.get(i);
		}
	}
	
	//使用FNV1_32_HASH算法计算服务器的Hash值,这里不使用重写hashCode的方法，最终效果没区别
	private static int getHash(String str) {
		final int p = 16777619;
		int hash = (int) 2166136261L;
		for (int i = 0; i < str.length(); i++)
			hash = (hash ^ str.charAt(i)) * p;
		hash += hash << 13;
		hash ^= hash >> 7;
		hash += hash << 3;
		hash ^= hash >> 17;
		hash += hash << 5;
 
		// 如果算出来的值为负数则取其绝对值
		if (hash < 0)
			hash = Math.abs(hash);
		return hash;
		}
 
	public static void main(String[] args) {
		String[] keys = {"太阳", "月亮", "星星"};
		for(int i=0; i<keys.length; i++)
			System.out.println("[" + keys[i] + "]的hash值为" + getHash(keys[i])
					+ ", 被路由到结点[" + getServer(keys[i]) + "]");
	}
}
```

#### **2、带虚拟节点的**

``` java

package hash;
 
import java.util.LinkedList;
import java.util.List;
import java.util.SortedMap;
import java.util.TreeMap;
 
import org.apache.commons.lang.StringUtils;
 
/**
  * 带虚拟节点的一致性Hash算法
  */
 public class ConsistentHashingWithoutVirtualNode {
 
     //待添加入Hash环的服务器列表
     private static String[] servers = {"192.168.0.0:111", "192.168.0.1:111", "192.168.0.2:111",
             "192.168.0.3:111", "192.168.0.4:111"};
     
     //真实结点列表,考虑到服务器上线、下线的场景，即添加、删除的场景会比较频繁，这里使用LinkedList会更好
     private static List<String> realNodes = new LinkedList<String>();
     
     //虚拟节点，key表示虚拟节点的hash值，value表示虚拟节点的名称
     private static SortedMap<Integer, String> virtualNodes = new TreeMap<Integer, String>();
             
     //虚拟节点的数目，这里写死，为了演示需要，一个真实结点对应5个虚拟节点
     private static final int VIRTUAL_NODES = 5;
     
     static{
         //先把原始的服务器添加到真实结点列表中
         for(int i=0; i<servers.length; i++)
             realNodes.add(servers[i]);
         
         //再添加虚拟节点，遍历LinkedList使用foreach循环效率会比较高
         for (String str : realNodes){
             for(int i=0; i<VIRTUAL_NODES; i++){
                 String virtualNodeName = str + "&&VN" + String.valueOf(i);
                 int hash = getHash(virtualNodeName);
                 System.out.println("虚拟节点[" + virtualNodeName + "]被添加, hash值为" + hash);
                 virtualNodes.put(hash, virtualNodeName);
             }
         }
         System.out.println();
     }
     
     //使用FNV1_32_HASH算法计算服务器的Hash值,这里不使用重写hashCode的方法，最终效果没区别
     private static int getHash(String str){
         final int p = 16777619;
         int hash = (int)2166136261L;
         for (int i = 0; i < str.length(); i++)
             hash = (hash ^ str.charAt(i)) * p;
         hash += hash << 13;
         hash ^= hash >> 7;
         hash += hash << 3;
         hash ^= hash >> 17;
         hash += hash << 5;
         
         // 如果算出来的值为负数则取其绝对值
         if (hash < 0)
             hash = Math.abs(hash);
         return hash;
     }
     
     //得到应当路由到的结点
     private static String getServer(String key){
        //得到该key的hash值
         int hash = getHash(key);
         // 得到大于该Hash值的所有Map
         SortedMap<Integer, String> subMap = virtualNodes.tailMap(hash);
         String virtualNode;
         if(subMap.isEmpty()){
            //如果没有比该key的hash值大的，则从第一个node开始
            Integer i = virtualNodes.firstKey();
            //返回对应的服务器
            virtualNode = virtualNodes.get(i);
         }else{
            //第一个Key就是顺时针过去离node最近的那个结点
            Integer i = subMap.firstKey();
            //返回对应的服务器
            virtualNode = subMap.get(i);
         }
         //virtualNode虚拟节点名称要截取一下
         if(StringUtils.isNotBlank(virtualNode)){
             return virtualNode.substring(0, virtualNode.indexOf("&&"));
         }
         return null;
     }
     
     public static void main(String[] args){
         String[] keys = {"太阳", "月亮", "星星"};
         for(int i=0; i<keys.length; i++)
             System.out.println("[" + keys[i] + "]的hash值为" +
                     getHash(keys[i]) + ", 被路由到结点[" + getServer(keys[i]) + "]");
     }
 }
```

