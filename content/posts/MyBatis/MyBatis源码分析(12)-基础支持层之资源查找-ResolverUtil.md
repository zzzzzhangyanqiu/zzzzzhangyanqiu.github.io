---
title: MyBatis源码分析(12)-基础支持层之资源查找-ResolverUtil&&VFS
tags:
  - MyBatis
date: 2018-11-18
categories:
 - MyBatis
---

## 资源查找

本篇文章主要讲解`MyBatis`中的两个类`VFS`、`ResolverUtil`，由于这两个类功能类似，都被用于资源查找，所以放在同一篇文章中讲解。

## ResolverUtil

`ResolverUtil`提供了根据特定条件搜索指定包的功能。



该类实现比较简单，首先一起来看如何使用。如果我们想查找`org.apache.ibatis.annotations`下`Arg`的子类，则

```java
ResolverUtil resolverUtil = new ResolverUtil();
resolverUtil.find(new ResolverUtil.IsA(Arg.class),"org.apache.ibatis.annotations");
Set classes = resolverUtil.getClasses();
```



`ResolverUtil`工具类中有两个主要字段

```java
/** 搜索到的符合条件的类 用来存储搜索到的结果 */
private Set<Class<? extends T>> matches = new HashSet<Class<? extends T>>();

/**
   * 查找类时使用的ClassLoader. 如果为空则使用Thread.currentThread().getContextClassLoader()
   */
private ClassLoader classloader;
```

### Test

`Test`是`ResolverUtil`的一个内部接口，指定如何搜索类。

```java
public static interface Test {
    /**
     * 匹配条件
     */
    boolean matches(Class<?> type);
}
```

`MyBatis`为`Test`接口提供了两个实现类，用户也可以自定义实现。两个实现类分别是`IsA`（查找指定类的子类）和`AnnotatedWith`（查找包含指定注解的类），实现如下

```java
public static class IsA implements Test {
    private Class<?> parent;

    /** 使用指定的父类来查找子类 */
    public IsA(Class<?> parentType) {
        this.parent = parentType;
    }

    /** 调用Class.isAssignableFrom()判断是否是指定类的子类 */
    @Override
    public boolean matches(Class<?> type) {
        return type != null && parent.isAssignableFrom(type);
    }

    //省略toString()
}


public static class AnnotatedWith implements Test {
    private Class<? extends Annotation> annotation;

    /** 使用指定的注解来匹配 */
    public AnnotatedWith(Class<? extends Annotation> annotation) {
        this.annotation = annotation;
    }

    /** 调用 Class.isAnnotationPresent() 判断是否包含指定注解*/
    @Override
    public boolean matches(Class<?> type) {
        return type != null && type.isAnnotationPresent(annotation);
    }

    //省略toString()
}

```

### find()

`find(Test test, String packageName)`是`ResolverUtil`中的核心方法，其它方法如`findAnnotated()/findImplementations()`也都是通过此方法实现的

```java
public ResolverUtil<T> find(Test test, String packageName) {
    //将packageName的“.”替换成“/”
    String path = getPackagePath(packageName);

    try {
        //查找该路径下的所有文件
        List<String> children = VFS.getInstance().list(path);
        for (String child : children) {
            //只处理class文件
            if (child.endsWith(".class")) {
                //如果匹配则添加到matches字段
                addIfMatching(test, child);
            }
        }
    } catch (IOException ioe) {
        log.error("...");
    }

    //链式调用
    return this;
}
```

```java
protected void addIfMatching(Test test, String fqn) {
    try {
        //将包路径中的'/'替换为'.'
        String externalName = fqn.substring(0, fqn.indexOf('.')).replace('/', '.');
        ClassLoader loader = getClassLoader();
        //省略日志记录

        //加载该类
        Class<?> type = loader.loadClass(externalName);

        //如果匹配则加入到matches集合中
        if (test.matches(type)) {
            matches.add((Class<T>) type);
        }
    } catch (Throwable t) {
        log.warn("...");
    }
}
```


## VFS

`VFS(Virtual File System)`表示虚拟文件系统，主要用来查找指定路径下的资源。`VFS`是一个抽象类，

首先看一下使用方式（这里使用的是`DefaultVFS`）

```java
VFS vfs;

@Before
public void before() {
    vfs = new DefaultVFS();
}

@Test
public void test() throws IOException {
    //在E:\mybatis\src\main\resources\mybatis.jar文件中
    //查找org/apache/ibatis/annotations下的文件
    String path = "org/apache/ibatis/annotations";
    URL url = new URL("file:\\E:\\mybatis\\src\\main\\resources\\mybatis.jar");
    List<String> list = vfs.list(url, path);
    list.forEach(System.out::println);
}
```

下面开始分析，此类的主要字段有

```java
/** MyBatis提供的两个VFS子类 */
public static final Class<?>[] IMPLEMENTATIONS = { JBoss6VFS.class, DefaultVFS.class };

/** 用户自定义的子类，可以通过VFS.addImplClass()加入此集合中 */
public static final List<Class<? extends VFS>> USER_IMPLEMENTATIONS = new ArrayList<Class<? extends VFS>>();

/** 当前使用的VFS对象 */
private static VFS instance;
```

### getInstance()

`VFS`使用单例模式来创建`instance`对象，如果你对该设计模式还不是很熟悉，建议先看一下[这边文章](https://suiyueranzly.gitee.io/posts/1776793711/)或者去网上找一些相关资料。`getInstance()`方法实现如下

```java
public static VFS getInstance() {
    //如果已经创建过了则直接返回
    if (instance != null) {
        return instance;
    }

    List<Class<? extends VFS>> impls = new ArrayList<Class<? extends VFS>>();
    //首先加入用户自定义的子类
    impls.addAll(USER_IMPLEMENTATIONS);
    //MyBatis中提供的子类
    impls.addAll(Arrays.asList((Class<? extends VFS>[]) IMPLEMENTATIONS));

    //遍历检查impls中每个子类，找到可用的一个
    VFS vfs = null;
    for (int i = 0; vfs == null || !vfs.isValid(); i++) {
        Class<? extends VFS> impl = impls.get(i);
        vfs = impl.newInstance();
    }
    
    //省略try-catch和日志记录

    VFS.instance = vfs;
    return VFS.instance;
}
```

`VFS`中有两个抽象方法需要子类实现

```java
/** 标记该类是否可用 */
public abstract boolean isValid();

/**
   * 查找资源	此方法是该类的核心方法
   */
protected abstract List<String> list(URL url, String forPath) throws IOException;
```

### DefaultVFS

`MyBatis`为`VFS`提供了两个子类，分别是`JBoss6VFS`和 `DefaultVFS`，下面以`DefaultVFS`为例进行讲解，有兴趣的同学可自行学习`JBoss6VFS`类。

#### list()

`DefaultVFS.list()`实现如下（代码中已忽略日志记录操作）

```java
public List<String> list(URL url, String path) throws IOException {
    InputStream is = null;
    try {
        List<String> resources = new ArrayList<String>();

        // 调用findJarForResource检测url，如果是指定资源是jar包则返回原url
        // 如果不是jar包则返回null
        URL jarUrl = findJarForResource(url);
        //如果是jar包
        if (jarUrl != null) {
            is = jarUrl.openStream();
            //查找jar包内path下的文件
            resources = listResources(new JarInputStream(is), path);
        }
        else {
            List<String> children = new ArrayList<String>();
            try {
                //即使给定的资源不是jar包，有的时候也可以通过JarInputStream打开，这里再次做校验
                if (isJar(url)) {
                    is = url.openStream();
                    JarInputStream jarInput = new JarInputStream(is);
                    //遍历jar包内文件加入到children集合中
                    for (JarEntry entry; (entry = jarInput.getNextJarEntry()) != null;) {
                        children.add(entry.getName());
                    }
                    jarInput.close();
                }
                else {
                    /*
                     * 该else分支主要处理文件，但是有一些servlet容器允许从目录资源中读取内容，
                     * 所以这里就不会抛出FileNotFoundException异常，
                      * 而且没有提供通过读取内容来区分是文件还是目录的方法
                      * 为了解决这个问题，在读取每一行时，尝试通过ClassLoader来加载此资源
                      * 只要加载的内容是空，则当前资源就是目录
                     */
                    is = url.openStream();
                    BufferedReader reader = new BufferedReader(new InputStreamReader(is));
                    List<String> lines = new ArrayList<String>();
                    for (String line; (line = reader.readLine()) != null;) {
                        lines.add(line);
                        if (getResources(path + "/" + line).isEmpty()) {
                            lines.clear();
                            break;
                        }
                    }
                    //如果不是空，则将此文件加入到children集合中，
                    //稍后用来递归list()方法进行查找
                    if (!lines.isEmpty()) {
                        children.addAll(lines);
                    }
                }
            } catch (FileNotFoundException e) {

                /*
            	* 抛出FileNotFoundException异常则证明传入的url是目录
            	* */
                if ("file".equals(url.getProtocol())) {
                    File file = new File(url.getFile());
                    //如果是目录则获取下面所有文件
                    if (file.isDirectory()) {
                        children = Arrays.asList(file.list());
                    }
                }
                else {
                    throw e;
                }
            }

            String prefix = url.toExternalForm();
            if (!prefix.endsWith("/")) {
                prefix = prefix + "/";
            }

            // 拼接子目录的路径
            for (String child : children) {
                String resourcePath = path + "/" + child;
                resources.add(resourcePath);
                URL childUrl = new URL(prefix + child);
                //递归查找所有子目录
                resources.addAll(list(childUrl, resourcePath));
            }
        }

        return resources;
    } finally {
        //关闭资源
    }
}

```

#### findJarForResource()

`findJarForResource()`方法用于检测传入的`url`，如果是指定资源是`jar`包则返回原`url`，如果不是`jar`包则返回`null`（代码中已忽略日志记录操作）

```java
protected URL findJarForResource(URL url) throws MalformedURLException {
    // 传入的URL文件名可能是一个URL，循环检测
    try {
        for (;;) {
            url = new URL(url.getFile());
        }
    } catch (MalformedURLException e) {
        // 这里做为for循环的弹出条件
    }

    // 寻找.jar的拓展名，如果有则删除后面所有内容，没有则直接返回null
    StringBuilder jarUrl = new StringBuilder(url.toExternalForm());
    int index = jarUrl.lastIndexOf(".jar");
    if (index >= 0) {
        jarUrl.setLength(index + 4);
    }
    else {
        //如果不包含".jar"则返回null
        return null;
    }

    // 打开文件并检测
    try {
        URL testUrl = new URL(jarUrl.toString());
        //检测是否是jar包
        if (isJar(testUrl)) {
            return testUrl;
        }
        else {
            // 如果不是，则检测该文件是否存在
            jarUrl.replace(0, jarUrl.length(), testUrl.getFile());
            File file = new File(jarUrl.toString());

            // 如果不存在，将文件名转码后重新检测
            if (!file.exists()) {
                try {
                    file = new File(URLEncoder.encode(jarUrl.toString(), "UTF-8"));
                } catch (UnsupportedEncodingException e) {
                    throw new RuntimeException("...");
                }
            }
            
            //如果存在则检测是否是jar包
            if (file.exists()) {
                testUrl = file.toURI().toURL();
                if (isJar(testUrl)) {
                    return testUrl;
                }
            }
        }
    } catch (MalformedURLException e) {
        log.warn("...");
    }
    return null;
}
```

#### isJar()

`isJar`方法用来检测指定的文件是否是`jar`包格式，通过读取文件前四位进行检测。（代码中已忽略日志记录操作）

```java
protected boolean isJar(URL url, byte[] buffer) {
    InputStream is = null;
    is = url.openStream();
    //读取文件前四位进行判断
    //如果前四位为'P', 'K', 3, 4则认定此文件是jar包
    is.read(buffer, 0, JAR_MAGIC.length);
    // private static final byte[] JAR_MAGIC = { 'P', 'K', 3, 4 };
    if (Arrays.equals(buffer, JAR_MAGIC)) {
        return true;
    }
	//省略try-catch
    return false;
}
```



#### listResources()

`listResources()`主要用来查找`jar`包中指定`path`的文件。（代码中已忽略日志记录操作）

```java
protected List<String> listResources(JarInputStream jar, String path) throws IOException {
    // 在前后拼接"/"
    if (!path.startsWith("/")) {
        path = "/" + path;
    }
    if (!path.endsWith("/")) {
        path = path + "/";
    }

    // 读取jar包中的每一个文件，判断是否是给定的path
    List<String> resources = new ArrayList<String>();
    for (JarEntry entry; (entry = jar.getNextJarEntry()) != null;) {
        if (!entry.isDirectory()) {
            String name = entry.getName();
            if (!name.startsWith("/")) {
                name = "/" + name;
            }
            if (name.startsWith(path)) {
                resources.add(name.substring(1));
            }
        }
    }
    return resources;
}
```















