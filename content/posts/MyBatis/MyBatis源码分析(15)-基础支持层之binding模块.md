---
title: MyBatis源码分析(15)-基础支持层之binding模块
tags:
  - MyBatis
date: 2018-12-30
categories:
 - MyBatis
---

## binding模块

我们在使用`MyBatis`的时候，通常都是定义一个接口，然后使用`xml`或者注解的方式来编辑`SQL`语句，查询数据库时直接调用此接口就可以了。注意，此时的接口没有实现类，那么`MyBatis`是通过什么方式来实现接口和`xml`文件绑定的呢？今天我们就一起来了解`MyBatis`的`binding`模块。了解此模块之前，首先来看一下此模块的整体结构图

![](/images/oss/%E5%8D%9A%E5%AE%A2/binding%E6%A8%A1%E5%9D%97%E7%B1%BB%E7%BB%93%E6%9E%84%E5%9B%BE.png)

此模块的主要思路为：`MyBatis`初始化时，会为接口（以下统称为`mapper`接口）创建代理类，当用户调用接口中的方法时，最终会执行代理类中的方法从而实现增删改查。

### MapperRegistry&&MapperProxyFactory

`MapperRegistry`主要用来注册`mapper`接口和其对应的`MapperProxyFactory`，而`MapperProxyFactory`负责创建代理类。`MapperRegistry`字段如下

```java
/*** MyBatis配置 **/
private final Configuration config;
/*** 存储mapper接口和其对应的MapperProxyFactory ***/
private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<Class<?>, MapperProxyFactory<?>>();
```

`MyBatis`初始化时会调用`MapperRegistry.addMapper()`来注册`mapper`（也就是填充`knownMappers`集合），方法实现如下

```java
/**
   * 方法参数为mapper接口的class对象
   * **/
public <T> void addMapper(Class<T> type) {
    //判断是否是接口
    if (type.isInterface()) {
        //如果已经注册过了，就抛出异常
        if (hasMapper(type)) {
            throw new BindingException(".....");
        }
        //是否注册完毕
        boolean loadCompleted = false;
        try {
            //首先将其加入到knownMappers集合中
            knownMappers.put(type, new MapperProxyFactory<T>(type));
            // 处理XML和注解
            MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
            parser.parse();
            //如果解析完成之后没有抛出异常，则证明添加当前接口成功
            loadCompleted = true;
        } finally {
            //如果解析失败则移除之前添加的接口
            if (!loadCompleted) {
                knownMappers.remove(type);
            }
        }
    }
}

public <T> boolean hasMapper(Class<T> type) {
    return knownMappers.containsKey(type);
}
```

从`MyBatis3.2.2`版本开始，开始支持批量注册

```java
public void addMappers(String packageName) {
    addMappers(packageName, Object.class);
}

public void addMappers(String packageName, Class<?> superType) {
    //使用ResolverUtil解析指定包下指定类的子类
    //ResolverUtil工具类之前的文章已经分析过
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<Class<?>>();
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    Set<Class<? extends Class<?>>> mapperSet = resolverUtil.getClasses();
    //循环调用addMapper方法
    for (Class<?> mapperClass : mapperSet) {
        addMapper(mapperClass);
    }
}
```

`MapperRegistry.getMapper()`用来获取`mapper`接口对应的代理对象

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    //取出mapper接口对应的MapperProxyFactory对象
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
        throw new BindingException("....");
    }
    try {
        //使用动态代理实例化
        return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
        throw new BindingException("....");
    }
}
```

`MapperProxyFactory`负责创建对应`mapper`接口的代理对象，该类的字段有

```java
/** 当前对象的对应接口 **/
private final Class<T> mapperInterface;
/** 因为一个接口中可能有多个方法
*  这里每个方法对应一个MapperMethod对象
* **/
private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();
```

`MapperProxyFactory.newInstance()`会创建指定接口的动态代理对象

```java
protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}

public T newInstance(SqlSession sqlSession) {
    //MapperProxy就是接口的动态代理对象
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
}
```

### MapperProxy

`MapperProxy`做为`mapper`接口的动态代理对象，其字段有

```java
/** 序列化 **/
private static final long serialVersionUID = -6424540398559729838L;
/** 对应的sqlsession对象 **/
private final SqlSession sqlSession;
/** 对应的mapper接口 **/
private final Class<T> mapperInterface;
/** 接口中 方法和MapperMethod的对应 **/
private final Map<Method, MapperMethod> methodCache;
```

既然是动态代理，那么核心方法就肯定是`invoke()`了

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        //如果调用的是Object的方法，如toString()，则直接调用
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);
        } else if (isDefaultMethod(method)) {
            //处理Java7以上版本的动态类型语言
            return invokeDefaultMethod(proxy, method, args);
        }
    } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
    }
    //获取指定方法对应的MapperMethod
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
}
```

`MapperProxy.cachedMapperMethod()`方法会从`methodCache`集合中获取值，如果没有则实例化后填充到`methodCache`集合。

```java
private MapperMethod cachedMapperMethod(Method method) {
    //获取MapperMethod对象
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
        //如果为空，则实例化后放入methodCache集合
        mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
        methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
}
```

### MapperMethod

通过之前的介绍我们可以知道，调用`mapper`接口中的方法时，因为动态代理模式，最终会调用到`MapperMethod`对象上。该类记录了方法信息、`SQL`语句信息等。此对象可以看作是`mappper`接口中的方法和`SQL`语句的桥梁。

首先来下一下该类的字段

```java
/*** 记录了SQL语句的名称和类型 **/
private final SqlCommand command;
/*** 记录方法的返回类型、参数等信息 **/
private final MethodSignature method;
```

`SqlCommand`和`MethodSignature`都是`MapperMethod`的内部类，在`MapperMethod`的构造方法中会初始化两个类

```java
public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    this.command = new SqlCommand(config, mapperInterface, method);
    this.method = new MethodSignature(config, mapperInterface, method);
}
```

#### SqlCommand

`SqlCommand`类主要记录了`SQL`语句相关的信息

```java
/** 记录了SQL语句的名称 **/
private final String name;
/** 记录了SQL语句的类型 **/
private final SqlCommandType type;
```

`SqlCommandType`类是一个枚举类型，其中定义了`SQL`的类型，如`INSERT, UPDATE, DELETE..`等等

`SqlCommand`中没有其它方法，上述两个字段会在类初始化时被赋值

```java
public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
    //获取方法的签名 如UserMapper.getUsers
    String statementName = mapperInterface.getName() + "." + method.getName();
    //MappedStatement记录了sql语句的信息，会在MyBatis初始化时创建，后面会详细描述
    MappedStatement ms = null;
    //如果存在此对象则直接取出
    if (configuration.hasStatement(statementName)) {
        ms = configuration.getMappedStatement(statementName);
    } else if (!mapperInterface.equals(method.getDeclaringClass())) {
        //处理  方法在父类中声明的情况
        String parentStatementName = method.getDeclaringClass().getName() + "." + method.getName();
        if (configuration.hasStatement(parentStatementName)) {
            ms = configuration.getMappedStatement(parentStatementName);
        }
    }
    //如果MappedStatement为空，判断是否有flush注解，如果没有直接抛出异常
    if (ms == null) {
        if(method.getAnnotation(Flush.class) != null){
            name = null;
            type = SqlCommandType.FLUSH;
        } else {
            throw new BindingException("....");
        }
    } else {
        //不为空则获取SQL名称和类型
        name = ms.getId();
        type = ms.getSqlCommandType();
        if (type == SqlCommandType.UNKNOWN) {
            throw new BindingException("...");
        }
    }
}
```

#### MethodSignature

`MethodSignature`类记录了方法相关的信息

```java
/** 返回值类型是否是集合 **/
private final boolean returnsMany;
/** 返回值类型是否是map **/
private final boolean returnsMap;
/** 返回值类型是否是void **/
private final boolean returnsVoid;
/** 返回值类型是否是cursor **/
private final boolean returnsCursor;
/** 返回值类型 **/
private final Class<?> returnType;
/** 如果返回值类型是map，此处保存做为map-key的名称 **/
private final String mapKey;
/** 方法参数列表中，resultHandler的位置 **/
private final Integer resultHandlerIndex;
/** 方法参数列表中，rowBounds的位置 **/
private final Integer rowBoundsIndex;
/** 该方法对应的ParamNameResolver对象  稍后会做分析 **/
private final ParamNameResolver paramNameResolver;
```

`MethodSignature`构造方法中会对上述字段进行赋值

```java
public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
    //TypeParameterResolver.resolveReturnType()用来解析方法返回值类型，之前文章有讲过
    Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
    if (resolvedReturnType instanceof Class<?>) {
        this.returnType = (Class<?>) resolvedReturnType;
    } else if (resolvedReturnType instanceof ParameterizedType) {
        //如果是泛型类型，则取出原始类型
        this.returnType = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
    } else {
        this.returnType = method.getReturnType();
    }
    this.returnsVoid = void.class.equals(this.returnType);
    this.returnsMany = (configuration.getObjectFactory().isCollection(this.returnType) || this.returnType.isArray());
    this.returnsCursor = Cursor.class.equals(this.returnType);
    //获取做为map-key的名称
    this.mapKey = getMapKey(method);
    this.returnsMap = (this.mapKey != null);
    //getUniqueParamIndex()用来获取参数列表中，指定参数的位置
    this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
    this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
    //解析方法参数，该类稍后讲解
    this.paramNameResolver = new ParamNameResolver(configuration, method);
}
```

`getMapKey()`方法逻辑比较简单，获取方法上`@MapKey`注解的值

```java
private String getMapKey(Method method) {
    String mapKey = null;
    //判断方法返回值是否是map
    if (Map.class.isAssignableFrom(method.getReturnType())) {
        //取出注解的值
        final MapKey mapKeyAnnotation = method.getAnnotation(MapKey.class);
        if (mapKeyAnnotation != null) {
            mapKey = mapKeyAnnotation.value();
        }
    }
    return mapKey;
}
```

`getUniqueParamIndex()`方法会从指定方法的参数列表中，找到指定参数的位置

```java
private Integer getUniqueParamIndex(Method method, Class<?> paramType) {
    Integer index = null;
    //获取方法的所有参数类型
    final Class<?>[] argTypes = method.getParameterTypes();
    for (int i = 0; i < argTypes.length; i++) {
        //找到对应的位置
        if (paramType.isAssignableFrom(argTypes[i])) {
            //一个方法的参数列表中，只能有一个指定类型
            if (index == null) {
                index = i;
            } else {
                throw new BindingException("...");
            }
        }
    }
    return index;
}
```

接下来我们一起来看`ParamNameResolver`

### ParamNameResolver

`ParamNameResolver`主要用来处理指定方法的参数，该类有两个主要字段，其中`SortedMap<Integer, String> names`用来存储方法参数索引和名称之间的对应关系，如`aMethod(@Param("M") int a, @Param("N") int b)`方法会解析成`{0, "M"}, {1, "N"}`，如果未使用`@Param`注解来指定名称时，使用的是参数索引，如`aMethod(int a, int b)`方法会被解析成`{0, "0"}, {1, "1"}`。需要注意的是，该类并不会处理两个特殊类型`RowBounds`和`ResultHandler`，如`aMethod(int a, RowBounds rb, int b)`方法会解析成`{0, "0"}, {2, "1"}`。首先来看一下主要字段

```java
/*** 该字段已经说过 **/
private final SortedMap<Integer, String> names;

/** 是否有@param注解 **/
private boolean hasParamAnnotation;
```

`ParamNameResolver`初始化时会进行参数的解析

```java
public ParamNameResolver(Configuration config, Method method) {
    //获取所有的参数类型
    final Class<?>[] paramTypes = method.getParameterTypes();
    //获取参数列表的注解
    final Annotation[][] paramAnnotations = method.getParameterAnnotations();
    final SortedMap<Integer, String> map = new TreeMap<Integer, String>();
    int paramCount = paramAnnotations.length;
    for (int paramIndex = 0; paramIndex < paramCount; paramIndex++) {
        //如果是特殊类型，如RowBounds、ResultHandler则不做处理
        if (isSpecialParameter(paramTypes[paramIndex])) {
            continue;
        }
        String name = null;
        //获取该参数的所有注解
        for (Annotation annotation : paramAnnotations[paramIndex]) {
            //如果有@param注解，则将其value做为参数名
            if (annotation instanceof Param) {
                hasParamAnnotation = true;
                name = ((Param) annotation).value();
                break;
            }
        }
        //如果没有@param注解
        if (name == null) {
            // 根据配置决定是否使用参数的真实名称做为参数名
            if (config.isUseActualParamName()) {
                name = getActualParamName(method, paramIndex);
            }
            //如果等于空则使用参数索引
            if (name == null) {
                name = String.valueOf(map.size());
            }
        }
        map.put(paramIndex, name);
    }
    //返回一个只读集合
    names = Collections.unmodifiableSortedMap(map);
}
```

`ParamNameResolver.getNamedParams(Object[] args)`方法会把传入的实参列表转换成参数名-参数实际值返回，返回类型为`ParamMap`，`ParamMap`继承了`HashMap`，除了在`get()`获取不到值时会抛异常，其余功能与`HashMap`相同。

```java
public Object getNamedParams(Object[] args) {
    final int paramCount = names.size();
    if (args == null || paramCount == 0) {
        return null;
    } else if (!hasParamAnnotation && paramCount == 1) {
        //如果没有@param注解且只有一个参数则直接返回
        return args[names.firstKey()];
    } else {
        final Map<String, Object> param = new ParamMap<Object>();
        int i = 0;
        //遍历，存储参数名和参数的实际值
        for (Map.Entry<Integer, String> entry : names.entrySet()) {
            param.put(entry.getValue(), args[entry.getKey()]);
            //添加通用参数名称  GENERIC_NAME_PREFIX="param"
            //将参数拼接成param1，param2等
            final String genericParamName = GENERIC_NAME_PREFIX + String.valueOf(i + 1);
            // 如果存在就不需要添加了
            if (!names.containsValue(genericParamName)) {
                param.put(genericParamName, args[entry.getKey()]);
            }
            i++;
        }
        return param;
    }
}
```

### MapperMethod

说完了`SqlCommand`和`MethodSignature`，我们继续说回到`MapperMethod`，之前根据源码我们可以知道，调用`mapper`接口的方法时，最终都会调用到`MapperMethod.execute()`方法上

```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    //判断此次执行的操作类型
    switch (command.getType()) {
        case INSERT: {
            //将实参数组转换为参数名-参数实际值
            Object param = method.convertArgsToSqlCommandParam(args);
            //对于增删改操作，需要返回受影响的行数
            result = rowCountResult(sqlSession.insert(command.getName(), param));
            break;
        }
        case UPDATE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.update(command.getName(), param));
            break;
        }
        case DELETE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.delete(command.getName(), param));
            break;
        }
        case SELECT:
            //如果是查询，判断该方法的返回值是void还是list、map等，分别执行不同的逻辑
            //判断是否使用了ResultHandler类
            if (method.returnsVoid() && method.hasResultHandler()) {
                executeWithResultHandler(sqlSession, args);
                result = null;
            } else if (method.returnsMany()) {
                result = executeForMany(sqlSession, args);
            } else if (method.returnsMap()) {
                result = executeForMap(sqlSession, args);
            } else if (method.returnsCursor()) {
                result = executeForCursor(sqlSession, args);
            } else {
                Object param = method.convertArgsToSqlCommandParam(args);
                result = sqlSession.selectOne(command.getName(), param);
            }
            break;
        case FLUSH:
            result = sqlSession.flushStatements();
            break;
        default:
            throw new BindingException("Unknown execution method for: " + command.getName());
    }
    //如果查询结果为null，并且方法的的返回值是基本类型，并且方法的返回值不是void
    //则抛出异常
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
        throw new BindingException("....");
    }
    return result;
}
```

`method.convertArgsToSqlCommandParam()`方法用来将传入的的实参数组转换为参数名-参数实际值，看到这里是不是很熟悉，没错，它就是直接调用的`ParamNameResolver.getNamedParams(Object[] args)`方法

```java
public Object convertArgsToSqlCommandParam(Object[] args) {
    return paramNameResolver.getNamedParams(args);
}
```

`rowCountResult()`用来返回受影响的行数，注意，`SQL`操作受影响的行数其实在`sqlSession.insert()`等方法中就已经返回了，这里主要做一次类型转换

```java
private Object rowCountResult(int rowCount) {
    final Object result;
    //如果方法返回值是void则直接返回null
    if (method.returnsVoid()) {
        result = null;
    } else if (Integer.class.equals(method.getReturnType()) || Integer.TYPE.equals(method.getReturnType())) {
        //根据方法返回值类型的不同分别强转
        result = rowCount;
    } else if (Long.class.equals(method.getReturnType()) || Long.TYPE.equals(method.getReturnType())) {
        result = (long)rowCount;
    } else if (Boolean.class.equals(method.getReturnType()) || Boolean.TYPE.equals(method.getReturnType())) {
        //如果是boolean类型，则返回true或者false
        result = rowCount > 0;
    } else {
        //如果方法返回值是其他类型，这里抛出异常
        throw new BindingException("....");
    }
    return result;
}
```

在`select`判断中，如果使用了`ResultHandler`类来处理结果集，要调用`executeWithResultHandler()`进行特殊的处理

```java
private void executeWithResultHandler(SqlSession sqlSession, Object[] args) {
    MappedStatement ms = sqlSession.getConfiguration().getMappedStatement(command.getName());
    //使用ResultHandler时必须要配置ResultMap
    if (void.class.equals(ms.getResultMaps().get(0).getType())) {
        throw new BindingException(".....");
    }
    //转换参数
    Object param = method.convertArgsToSqlCommandParam(args);
    //判断是否有RowBound执行不同的方法重载
    if (method.hasRowBounds()) {
        RowBounds rowBounds = method.extractRowBounds(args);
        sqlSession.select(command.getName(), param, rowBounds, method.extractResultHandler(args));
    } else {
        sqlSession.select(command.getName(), param, method.extractResultHandler(args));
    }
}
```

在`select`判断中，如果返回值是集合，会调用`executeForMany()/executeForMap()`等方法进行结果集的处理，由于每个方法实现逻辑类似，这里只分析`executeForMany()`

```java
private <E> Object executeForMany(SqlSession sqlSession, Object[] args) {
    List<E> result;
    //参数转换
    Object param = method.convertArgsToSqlCommandParam(args);
    //判断是否有RowBound来执行不同的方法重载
    if (method.hasRowBounds()) {
        RowBounds rowBounds = method.extractRowBounds(args);
        result = sqlSession.<E>selectList(command.getName(), param, rowBounds);
    } else {
        result = sqlSession.<E>selectList(command.getName(), param);
    }
    // 判断方法的返回值是否符合结果集的类型
    if (!method.getReturnType().isAssignableFrom(result.getClass())) {
        //根据返回值类型分别处理
        if (method.getReturnType().isArray()) {
            return convertToArray(result);
        } else {
            return convertToDeclaredCollection(sqlSession.getConfiguration(), result);
        }
    }
    return result;
}
```

`convertToArray()`和`convertToDeclaredCollection()`方法功能类似，是将结果集转换为`array`数组或者`collection`

```java
private <E> Object convertToArray(List<E> list) {
    //返回表示数组的组件类型的Class。 如果此类型不是数组类，则返回null
    Class<?> arrayComponentType = method.getReturnType().getComponentType();
    //创建新数组
    Object array = Array.newInstance(arrayComponentType, list.size());
    //填充新数组
    if (arrayComponentType.isPrimitive()) {
        for (int i = 0; i < list.size(); i++) {
            Array.set(array, i, list.get(i));
        }
        return array;
    } else {
        return list.toArray((E[])array);
    }
}

private <E> Object convertToDeclaredCollection(Configuration config, List<E> list) {
    //根据方法返回值类型创建collection
    Object collection = config.getObjectFactory().create(method.getReturnType());
    MetaObject metaObject = config.newMetaObject(collection);
    //填充collection
    metaObject.addAll(list);
    return collection;
}
```









