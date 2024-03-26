---
title: Mybatis原理-动态代理
date: 2024-03-26 21:10:50
categories: 算法
top: 10
---


回想一下咱使用mybatis的时候，只定义了操作表的接口以及方法，并没有定义接口的实现类，一行jdbc代码都没写，就可以对数据库增删查改。哪有什么岁月静好，不过是有人替你负重前行。是勤恳的mybatis帮我们做了实现类，写了jdbc代码，并把mapper里的sql注入进去了。

梳理一下，框架的使用者需要做的事情包括：

1. 定义表的接口（如ActorPublicNoticeDAO）。
2. 定义表接口对应的XML，XML中包含查询SQL所需的参数，如表名、字段名等。
3. 调用表的接口方法，无需实现接口，直接调用即可。



框架的做的事包括：

1. 解析使用者定义的接口和XML配置。
2. 实现接口方法，在方法内部加入具体的JDBC执行代码，根据XML配置获取参数设置到JDBC执行代码中。



现在有一个很明显的问题，先现有框架，后有使用者，写框架的时候还不知道有什么接口，怎么写接口的实现类？

答：其实很简单，不知道有什么接口就约定一个指定路径，路径下的接口都扫描出来。

那怎么写接口的实现类呢？虽然写框架的时候不知道有什么接口，但是调用的时候就知道了呀。这不由让人想起来一句典型的定义：一种在运行时创建代理对象的机制，而不是在编译时手动编写代理类，这就是**动态代理**技术。



要使用Java动态代理，需要做以下几个步骤：

一、创建接口和方法。在这个项目里就是ActorPublicNoticeMapper#getInfoById。

```java

public interface ActorPublicNoticeMapper {
    String getInfoById(Integer id);
}

```



二、使用Proxy.newProxyInstance()方法创建接口的实例（即代理对象）。咱定义一个SqlSession类专门做这件事，你给它一个接口mapper，它给你返回一个mapper实例。

```java
public class SqlSession {
    /**
     * 返回代理类
     *
     * @param clazz
     * @param dbConfig
     * @param mapperBean
     * @return : T
     */
    public <T> T getMapper(Class<T> clazz, DbConfig dbConfig, MapperBean mapperBean) {
        return (T) Proxy.newProxyInstance(clazz.getClassLoader(), new Class[]{clazz}, new MyInvocationHandler(dbConfig, mapperBean));
    }

}
```



三、创建一个实现InvocationHandler接口的类，调用接口时，实际上是调用InvocationHandler中的invoke()方法。在这个项目里就是MyInvocationHandler。

```java

public class MyInvocationHandler implements InvocationHandler {

    private DbConfig dbConfig;
    private MapperBean mapperBean;


    public MyInvocationHandler(final DbConfig dbConfig, final MapperBean mapperBean) {
        this.dbConfig = dbConfig;
        this.mapperBean = mapperBean;
    }


    @Override
    public Object invoke(final Object proxy, final Method method, final Object[] args)  {
        try (Connection conn = DriverManager.getConnection(this.dbConfig.getUrl(), this.dbConfig.getUsername(), this.dbConfig.getPassword())) {
            Statement stmt = conn.createStatement();
            Map<String, MySqlFunction> functionMap = this.mapperBean.getFunctionMap();
            MySqlFunction mySqlFunction = functionMap.get(method.getName());
            String sql = parseSql(mySqlFunction.getSql(), args);
            ResultSet rs = stmt.executeQuery(sql);

            while (rs.next()){
                // 处理结果集
                return rs.getString("actorId");
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }


    public String parseSql(String sql, Object[] args) {
				//do something...
    }


}

```



Proxy.newProxyInstance()底层到底做了什么事呢，点进去看看这个方法吧。

```java
  public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * Look up or generate the designated proxy class.
         */
         //这里生成了一个类
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }

            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            //利用反射生产了这个类的对象
            return cons.newInstance(new Object[]{h});
        } 
```

在编译时不知道类还需要构造一个类的对象好说，用反射生成。那么是怎么生成类的呢。一般的类都是我们编译前就写好的.java文件，jvm把.java文件编译成字节码，并以.class文件存储。而动态代理是在运行时直接生成了.class文件。让我们点进代码一探究竟。

分别跳入如下方法：

java.lang.reflect.Proxy#newProxyInstance

java.lang.reflect.Proxy#getProxyClass0

java.lang.reflect.WeakCache#get

这里的代码比较绕，加入了一些缓存相关的处理，我们只需要抓住核心流程。

java.lang.reflect.Proxy.ProxyClassFactory#apply 这个方法就是动态代理生成字节码的核心代码，可以看到生成一个类，并且生成了3个方法。

```
private static final class ProxyClassFactory
        implements BiFunction<ClassLoader, Class<?>[], Class<?>>
    {
        // prefix for all proxy class names
        private static final String proxyClassNamePrefix = "$Proxy";

        // next number to use for generation of unique proxy class names
        private static final AtomicLong nextUniqueNumber = new AtomicLong();

        @Override
        public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

            Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
            for (Class<?> intf : interfaces) {
                /*
                 * Verify that the class loader resolves the name of this
                 * interface to the same Class object.
                 */
                Class<?> interfaceClass = null;
                try {
                    interfaceClass = Class.forName(intf.getName(), false, loader);
                } catch (ClassNotFoundException e) {
                }
                if (interfaceClass != intf) {
                    throw new IllegalArgumentException(
                        intf + " is not visible from class loader");
                }
                /*
                 * Verify that the Class object actually represents an
                 * interface.
                 */
                if (!interfaceClass.isInterface()) {
                    throw new IllegalArgumentException(
                        interfaceClass.getName() + " is not an interface");
                }
                /*
                 * Verify that this interface is not a duplicate.
                 */
                if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                    throw new IllegalArgumentException(
                        "repeated interface: " + interfaceClass.getName());
                }
            }

            String proxyPkg = null;     // package to define proxy class in
            int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

            /*
             * Record the package of a non-public proxy interface so that the
             * proxy class will be defined in the same package.  Verify that
             * all non-public proxy interfaces are in the same package.
             */
            for (Class<?> intf : interfaces) {
                int flags = intf.getModifiers();
                if (!Modifier.isPublic(flags)) {
                    accessFlags = Modifier.FINAL;
                    String name = intf.getName();
                    int n = name.lastIndexOf('.');
                    String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                    if (proxyPkg == null) {
                        proxyPkg = pkg;
                    } else if (!pkg.equals(proxyPkg)) {
                        throw new IllegalArgumentException(
                            "non-public interfaces from different packages");
                    }
                }
            }

            if (proxyPkg == null) {
                // if no non-public proxy interfaces, use com.sun.proxy package
                proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
            }

            /*
             * Choose a name for the proxy class to generate.
             */
            long num = nextUniqueNumber.getAndIncrement();
            String proxyName = proxyPkg + proxyClassNamePrefix + num;

            /*
             * Generate the specified proxy class.
             */
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            try {
                return defineClass0(loader, proxyName,
                                    proxyClassFile, 0, proxyClassFile.length);
            } catch (ClassFormatError e) {
                /*
                 * A ClassFormatError here means that (barring bugs in the
                 * proxy class generation code) there was some other
                 * invalid aspect of the arguments supplied to the proxy
                 * class creation (such as virtual machine limitations
                 * exceeded).
                 */
                throw new IllegalArgumentException(e.toString());
            }
        }
    }
```





