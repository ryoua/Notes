---
tag:
- 未看
flag: yellow
---
# Mybatis中的事务管理器详述

[原文链接](https://blog.csdn.net/andamajing/article/details/72026693)

在上篇文章<[Mybatis中的数据源与连接池详解 ](http://blog.csdn.net/majinggogogo/article/details/71715846)>中，我们结合源码对mybatis中的数据源和连接池进行了比较详细的说明。在这篇文章中，我们讲讲相关的另外一个主题——事务管理器。

在前面的文章中，我们知道mybatis支持两种事务类型，分别为JdbcTransaction和ManagedTransaction。接下来，我们从mybatis的xml配置文件入手，讲解事务管理器工厂的创建，然后讲述事务的创建和使用，最后分析这两种事务的实现和两者的区别。

我们先看看配置文件中相关的配置： 

![](https://img-blog.csdn.net/20170514164939290?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbWFqaW5nZ29nb2dv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

Mybatis定义了一个事务类型接口Transaction，JdbcTransaction和ManagedTransaction两种事务类型都实现了Transaction接口。我们看看Transaction这个接口的定义：

```
public interface Transaction {   /**   * Retrieve inner database connection   * @return DataBase connection   * @throws SQLException   */  Connection getConnection() throws SQLException;   /**   * Commit inner database connection.   * @throws SQLException   */  void commit() throws SQLException;   /**   * Rollback inner database connection.   * @throws SQLException   */  void rollback() throws SQLException;   /**   * Close inner database connection.   * @throws SQLException   */  void close() throws SQLException;   /**   * Get transaction timeout if set   * @throws SQLException   */  Integer getTimeout() throws SQLException;  }
```

 在事务接口中，定义了若干方法，如下结构所示：

![](https://img-blog.csdn.net/20170514165238385?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbWFqaW5nZ29nb2dv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

事务的继承关系如下：

![](https://img-blog.csdn.net/20170514165311260?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbWFqaW5nZ29nb2dv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

JdbcTransaction和ManagedTransaction的区别如下：

JdbcTransaction：利用java.sql.Connection对象完成对事务的提交（commit()）、回滚（rollback()）、关闭（close()）等；

ManagedTransaction：MyBatis自身不会去实现事务管理，而是让程序的容器来实现对事务的管理；

那么在mybatis中又是怎么使用事务管理器的呢？首先需要根据xml中的配置确定需要创建什么样的事务管理器，然后从事务管理器中获取相应的事务。

在mybatis初始化的时候，在解析<transactionManager>节点的时候，根据设置的type类型去初始化相应的事务管理器，解析源码如下所示：

```
 /**    * 解析<transactionManager>节点，创建对应的TransactionFactory    * @param context    * @return    * @throws Exception    */  private TransactionFactory transactionManagerElement(XNode context) throws Exception {    if (context != null) {      String type = context.getStringAttribute("type");      Properties props = context.getChildrenAsProperties();      /*           在Configuration初始化的时候，会通过以下语句，给JDBC和MANAGED对应的工厂类           typeAliasRegistry.registerAlias("JDBC", JdbcTransactionFactory.class);           typeAliasRegistry.registerAlias("MANAGED", ManagedTransactionFactory.class);           下述的resolveClass(type).newInstance()会创建对应的工厂实例      */      TransactionFactory factory = (TransactionFactory) resolveClass(type).newInstance();      factory.setProperties(props);      return factory;    }    throw new BuilderException("Environment declaration requires a TransactionFactory.");  }  
```

 从代码可以看出来，如果type配置成JDBC，则创建一个JdbcTransactionFactory实例，如果type配置成MANAGED，则会创建一个ManagedTransactionFactory实例。这两个事务管理器类型都实现了mybatis定义的TransactionFactory接口。

![](https://img-blog.csdn.net/20170514170734398?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbWFqaW5nZ29nb2dv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

事务管理器工厂接口的定义如下所示：

```
public interface TransactionFactory {   /**   * Sets transaction factory custom properties.   * @param props   */  void setProperties(Properties props);   /**   * Creates a {@link Transaction} out of an existing connection.   * @param conn Existing database connection   * @return Transaction   * @since 3.1.0   */  Transaction newTransaction(Connection conn);    /**   * Creates a {@link Transaction} out of a datasource.   * @param dataSource DataSource to take the connection from   * @param level Desired isolation level   * @param autoCommit Desired autocommit   * @return Transaction   * @since 3.1.0   */  Transaction newTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit); }
```

 从接口定义看，不管是JdbcTransactionFactory，还是ManagedTransactionFactory，都需要自行实现事务Transaction的创建工作。我们从源码上看看这两个类都是怎么定义实现的。

JdbcTransactionFactory定义如下： 

```
public class JdbcTransactionFactory implements TransactionFactory {   @Override  public void setProperties(Properties props) {  }   @Override  public Transaction newTransaction(Connection conn) {    return new JdbcTransaction(conn);  }   @Override  public Transaction newTransaction(DataSource ds, TransactionIsolationLevel level, boolean autoCommit) {    return new JdbcTransaction(ds, level, autoCommit);  }}
```

 ManagedTransactionFactory定义如下：

```
public class ManagedTransactionFactory implements TransactionFactory {   private boolean closeConnection = true;   @Override  public void setProperties(Properties props) {    if (props != null) {      String closeConnectionProperty = props.getProperty("closeConnection");      if (closeConnectionProperty != null) {        closeConnection = Boolean.valueOf(closeConnectionProperty);      }    }  }   @Override  public Transaction newTransaction(Connection conn) {    return new ManagedTransaction(conn, closeConnection);  }   @Override  public Transaction newTransaction(DataSource ds, TransactionIsolationLevel level, boolean autoCommit) {    // Silently ignores autocommit and isolation level, as managed transactions are entirely    // controlled by an external manager.  It's silently ignored so that    // code remains portable between managed and unmanaged configurations.    return new ManagedTransaction(ds, level, closeConnection);  }}
```

 从源码看，JdbcTransactionFactory会创建JDBC类型的事务JdbcTransaction，而ManagedTransactionFactory则会创建ManagedTransaction。

接下来，我们再来看看这两种类型的事务类型。

先从JdbcTransaction看起吧，这种类型的事务就是使用Java自带的Connection来实现事务的管理。connection对象的获取被延迟到调用getConnection()方法。如果autocommit设置为on，开启状态的话，它会忽略commit和rollback。直观地讲，就是JdbcTransaction是使用的java.sql.Connection 上的commit和rollback功能，JdbcTransaction只是相当于对java.sql.Connection事务处理进行了一次包装（wrapper），Transaction的事务管理都是通过java.sql.Connection实现的。

JdbcTransaction的代码实现如下：

```
public class JdbcTransaction implements Transaction {   private static final Log log = LogFactory.getLog(JdbcTransaction.class);   protected Connection connection;  protected DataSource dataSource;  protected TransactionIsolationLevel level;  protected boolean autoCommmit;   public JdbcTransaction(DataSource ds, TransactionIsolationLevel desiredLevel, boolean desiredAutoCommit) {    dataSource = ds;    level = desiredLevel;    autoCommmit = desiredAutoCommit;  }   public JdbcTransaction(Connection connection) {    this.connection = connection;  }   @Override  public Connection getConnection() throws SQLException {    if (connection == null) {      openConnection();    }    return connection;  }   @Override  public void commit() throws SQLException {    if (connection != null && !connection.getAutoCommit()) {      if (log.isDebugEnabled()) {        log.debug("Committing JDBC Connection [" + connection + "]");      }      connection.commit();    }  }   @Override  public void rollback() throws SQLException {    if (connection != null && !connection.getAutoCommit()) {      if (log.isDebugEnabled()) {        log.debug("Rolling back JDBC Connection [" + connection + "]");      }      connection.rollback();    }  }   @Override  public void close() throws SQLException {    if (connection != null) {      resetAutoCommit();      if (log.isDebugEnabled()) {        log.debug("Closing JDBC Connection [" + connection + "]");      }      connection.close();    }  }   protected void setDesiredAutoCommit(boolean desiredAutoCommit) {    try {      if (connection.getAutoCommit() != desiredAutoCommit) {        if (log.isDebugEnabled()) {          log.debug("Setting autocommit to " + desiredAutoCommit + " on JDBC Connection [" + connection + "]");        }        connection.setAutoCommit(desiredAutoCommit);      }    } catch (SQLException e) {      // Only a very poorly implemented driver would fail here,      // and there's not much we can do about that.      throw new TransactionException("Error configuring AutoCommit.  "          + "Your driver may not support getAutoCommit() or setAutoCommit(). "          + "Requested setting: " + desiredAutoCommit + ".  Cause: " + e, e);    }  }   protected void resetAutoCommit() {    try {      if (!connection.getAutoCommit()) {        // MyBatis does not call commit/rollback on a connection if just selects were performed.        // Some databases start transactions with select statements        // and they mandate a commit/rollback before closing the connection.        // A workaround is setting the autocommit to true before closing the connection.        // Sybase throws an exception here.        if (log.isDebugEnabled()) {          log.debug("Resetting autocommit to true on JDBC Connection [" + connection + "]");        }        connection.setAutoCommit(true);      }    } catch (SQLException e) {      if (log.isDebugEnabled()) {        log.debug("Error resetting autocommit to true "          + "before closing the connection.  Cause: " + e);      }    }  }   protected void openConnection() throws SQLException {    if (log.isDebugEnabled()) {      log.debug("Opening JDBC Connection");    }    connection = dataSource.getConnection();    if (level != null) {      connection.setTransactionIsolation(level.getLevel());    }    setDesiredAutoCommit(autoCommmit);  }   @Override  public Integer getTimeout() throws SQLException {    return null;  }  }
```

 最后我们再来看看ManagedTransaction对象，这个对象因为是将事务的管理交给容器去控制，所以，这里的ManagedTransaction是没有做任何控制的。我们先来看看源码：

```
public class ManagedTransaction implements Transaction {   private static final Log log = LogFactory.getLog(ManagedTransaction.class);   private DataSource dataSource;  private TransactionIsolationLevel level;  private Connection connection;  private boolean closeConnection;   public ManagedTransaction(Connection connection, boolean closeConnection) {    this.connection = connection;    this.closeConnection = closeConnection;  }   public ManagedTransaction(DataSource ds, TransactionIsolationLevel level, boolean closeConnection) {    this.dataSource = ds;    this.level = level;    this.closeConnection = closeConnection;  }   @Override  public Connection getConnection() throws SQLException {    if (this.connection == null) {      openConnection();    }    return this.connection;  }   @Override  public void commit() throws SQLException {    // Does nothing  }   @Override  public void rollback() throws SQLException {    // Does nothing  }   @Override  public void close() throws SQLException {    if (this.closeConnection && this.connection != null) {      if (log.isDebugEnabled()) {        log.debug("Closing JDBC Connection [" + this.connection + "]");      }      this.connection.close();    }  }   protected void openConnection() throws SQLException {    if (log.isDebugEnabled()) {      log.debug("Opening JDBC Connection");    }    this.connection = this.dataSource.getConnection();    if (this.level != null) {      this.connection.setTransactionIsolation(this.level.getLevel());    }  }   @Override  public Integer getTimeout() throws SQLException {    return null;  } }
```

 从源码我们可以看到，ManagedTransaction的commit和rollback方法是没有做任何事情的，它将事务交由了更上层的容易来进行控制和实现。至此，关于事务管理器我们描述的已经差不多了，如果需要深究可以自己再去研究研究。