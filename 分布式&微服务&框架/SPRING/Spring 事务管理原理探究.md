# [Spring 事务管理原理探究](https://www.cnblogs.com/duanxz/p/3750845.html)

此处先粘贴出Spring事务需要的配置内容：

1、Spring事务管理器的配置文件：

```
<bean id="transactionManager"  
class="org.springframework.jdbc.datasource.DataSourceTransactionManager">  
   <property name="dataSource" ref="dataSource" />.....  
</bean>
```

2、一个普通的JPA框架（此处是mybatis）的配置文件：    

```
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean"> 
   <property name="dataSource" ref="dataSource" />  
   .....         
</bean>
```

  这两个里面都配置了datasource，而且这个datasource的对象是在Spring的容器里面。一下提几个问题：

​            1、当JPA框架对数据库进行操作的时候，是从那里获取Connection？

​            2、jdbc对事务的配置，比如事务的开启，提交以及回滚是在哪里设置的？

​            3、Spring是通过aop拦截切面的所有需要进行事务管理的业务处理方法，那如何获取业务处理方法里面对数据库操作的事务呢？

​           现在我来对上面的问题来一一回答

​           1、这个问题很简单，既然在JPA的框架里面配置了datasource，那自然会从这个datasource里面去获得连接。

​           2、jdbc的事务配置是在Connection对像里面有对应的方法，比如setAutoCommit,commit,rollback这些方法就是对事务的操作。

​           3、Spring需要操作事务，那必须要对Connection来进行设置。Spring的AOP可以拦截业务处理方法，并且也知道业务处理方法里面的DAO操作的JAP框架是从datasource里面获取Connection对象，那么Spring需要对当前拦截的业务处理方法进行事务控制，那必然需要得到他内部的Connection对象。整体的结构图如下：

​           ![img](https://images0.cnblogs.com/i/285763/201405/251103539813827.jpg)

Spring 事务管理创造性的解决了很多以前要用重量级的应用服务器才能解决的事务问题，那么其实现原理一定很深奥吧？可是如果读者仔细研究了Spring事务管理的代码以后就会发现，事务管理其实也是如此简单的事情。这也印证了在本书开头的一句话“重剑无锋、大巧不工”，Spring并没有使用什么特殊的API，它运行的原理就是事务的原理。下面是DataSourceTransactionManager的启动事务用的代码（经简化）：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
protected void doBegin(Object transaction, TransactionDefinition definition)
{
 DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
 Connection con = null;
 try
 {
  if (txObject.getConnectionHolder() == null)
  {
     Connection newCon = this.dataSource.getConnection();
     txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
  }
  txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
  con = txObject.getConnectionHolder().getConnection();

  Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
  txObject.setPreviousIsolationLevel(previousIsolationLevel);
  if (con.getAutoCommit())
  {
    txObject.setMustRestoreAutoCommit(true);
    con.setAutoCommit(false);
  }
  txObject.getConnectionHolder().setTransactionActive(true);
  // Bind the session holder to the thread.
  if (txObject.isNewConnectionHolder())
  {
   TransactionSynchronizationManager.bindResource(getDataSource(),txObject.getConnectionHolder());
  }
 }
 catch (SQLException ex)
 {
  DataSourceUtils.releaseConnection(con, this.dataSource);
  throw new CannotCreateTransactionException( "Could not open JDBC Connection for transaction", ex);
 }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

在调用一个需要事务的组件的时候，管理器首先判断当前调用（即当前线程）有没有一个事务，如果没有事务则启动一个事务，并把事务与当前线程绑定。Spring使TransactionSynchronizationManager的bindResource方法将当前线程与一个事务绑定，采用的方式就是ThreadLocal，这可以从TransactionSynchronizationManager类的代码看出。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public abstract class TransactionSynchronizationManager 
{
 ……
 private static final ThreadLocal currentTransactionName = new ThreadLocal();
 private static final ThreadLocal currentTransactionReadOnly = new ThreadLocal();
 private static final ThreadLocal actualTransactionActive = new ThreadLocal(); ……
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

从doBegin的代码中可以看到在启动事务的时候，如果Connection是的自动提交的（也就是getAutoCommit()方法返回true）则事务管理就会失效，所以首先要调用setAutoCommit(false)方法将其改为非自动提交的。setAutoCommit(false)这个动作在有的JDBC驱动中会非常耗时，所以最好在配置数据源的时候就将“autoCommit”属性配置为true。