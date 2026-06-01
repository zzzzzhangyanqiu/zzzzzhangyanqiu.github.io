---
title: MyBatis源码分析(1)-基础支持层之解析器模块-XML解析
tags:
  - MyBatis
date: 2018-07-07
categories:
 - MyBatis
---
## 解析器模块 ##

`MyBatis`中涉及多个`XML`配置文件（如`mybatis-config.xml、mapper.xml`），因此我们从`XML`解析开始说起，比较常见解析`XML`方式主要有三种，分别是`DOM (Document Object Model）`解析方式和`SAX (Simple API for XML )`解析方式，以及从`JDK6.0` 版本开始支持的`StAX(Streaming API for XML)`，首先了解一下这几种解析方式。

### DOM ###

`DOM`解析方式是把整个`XML`文档加载到内存，并生成树结构，如下所示`inventory.xml`文件

```xml
	<?xml version="1.0" encoding="UTF-8"?>

	<inventory>
		<car year="2000">
			<name>blueCar</name>
			<color>blue</color>
			<price>14000</price>
			<speed>120</speed>
		</car>
		<car year="1995">
			<name>redCar</name>
			<color>red</color>
			<price>19000</price>
			<speed>150</speed>
		</car>
		<car year="2005">
			<name>blackCar</name>
			<color>black</color>
			<price>24000</price>
			<speed>170</speed>
		</car>
	</inventory>
```

经过`DOM`方式装载到内存后，会形成类似于如下结构


![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/8KoOAs9pwI.png)


相对于其它两种方式，此方式`API`使用简单，`MyBatis`正是采用了此种方式。由于是一次性把文档加载到内存中，如果文档较大，会比较浪费资源。

### SAX ###

`SAX`是基于事件的`XML`解析方式，它不会加载整个文档，而是只加载一部分就开始解析。开发人员可通过注册事件的方式来完成自定义操作，加载到某个节点时，会触发该节点注册的事件。

如果解析到某一处后，找到了符合条件的节点，那么就不需要再加载剩下的`XML`文件内容了。

`SAX`采用的是**推模式**，所谓的推模式就是由`SAX`来控制整个解析的流程，通过注册的事件推送给应用程序，应用程序无法控制整个解析流程，如下图

![image-20201229195617733](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/picgo/image-20201229195617733.png)

`SAX`的缺点也很明显，由于不加载整个`XML`文档，所以需要靠开发人员自己维护多级节点之间的关系，因为是流式处理（只能向后），所以`SAX`方式不能像`DOM`方式那样导航到之前节点。不支持`XPath`，不能提供写入`XML`文档的功能。

### StAX ###

`StAX `解析方式与`SAX `解析方式类似，它也是把`XML`文档作为一个事件流进行处理，但不同之处在于`StAX`采用的是**拉模式** 。所谓拉模式是应用程序通过调用解析器推进解析的进程，整个解析过程都是由应用程序来控制的。

![image-20201229195716521](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/picgo/image-20201229195716521.png)

### XPath简介 ###

> XPath即为XML路径语言（XML Path Language），它是一种用来确定XML文档中某部分位置的语言。

通过上面的描述可以得知，我们可以通过`XPath`在`XML`文档中找到特定的节点，下面是最常见的几个路径表达式。

| 表达式 | 描述 |
| :--------- | :----- |
| `nodename` | 选取此节点的所有子节点。 |
| `/` | 从根节点选取。 |
| `//` | 从匹配选择的当前节点选择文档中的节点，而不考虑它们的位置。 |
| `.` | 选取当前节点。 |
| `..` | 选取当前节点的父节点。 |
| `@` | 选取属性。 |
| `*` | 匹配任何元素节点。 |
| `@*` | 匹配任何属性节点。 |
| `node()` | 匹配任意类型的节点。 |
| `text()` | 匹配文本节点。 |
| `l` | 选取若干个节点。 |
| `[]` | 选取符合某些条件的节点。 |
下面是一个`JAVA`中解析上文中提到的`inventory.xml`并通过`XPath`查找节点的示例


```java
public static void main(String[] args) throws ParserConfigurationException, IOException, SAXException, XPathExpressionException {

    DocumentBuilderFactory documentBuilderFactory = DocumentBuilderFactory
            .newInstance();

    //校验文档
    documentBuilderFactory.setValidating(true);
    //提供对xml文件的支持
    documentBuilderFactory.setNamespaceAware(false);
    //忽略备注
    documentBuilderFactory.setIgnoringComments(true);
    //忽略空格
    documentBuilderFactory.setIgnoringElementContentWhitespace(false);
    //把CDATA节点转换为text节点
    documentBuilderFactory.setCoalescing(false);
    //展开实体引用节点
    documentBuilderFactory.setExpandEntityReferences(true);

    //创建文档解析器
    DocumentBuilder documentBuilder = documentBuilderFactory.newDocumentBuilder();

    documentBuilder.setErrorHandler(new ErrorHandler() {
        @Override
        public void warning(SAXParseException exception) throws SAXException {
            System.out.println("warning:" + exception.getMessage());
        }

        @Override
        public void error(SAXParseException exception) throws SAXException {
            System.out.println("error:" + exception.getMessage());
        }

        @Override
        public void fatalError(SAXParseException exception) throws SAXException {
            System.out.println("fatalError:" + exception.getMessage());
        }
    });

    //加载文档
    Document parse = documentBuilder.parse("src\\main\\resources\\inventory.xml");

    //创建xpathfactory

    XPathFactory factory = XPathFactory.newInstance();

    XPath xPath = factory.newXPath();

    //编译xpath表达式

    XPathExpression compile = xPath.compile("//car[color='black']/name/text()");

    //通过xpath表达式获得结果

    Object evaluate = compile.evaluate(parse, XPathConstants.NODESET);

    System.out.println("查询颜色是黑色的汽车");

    printNodeValue(parse, compile);

    System.out.println("查询价格小于于19000的汽车");
    
    compile = xPath.compile("//car[price < 19000]/name/text()");

    printNodeValue(parse, compile);

    System.out.println("查询1999年后所有汽车的属性和价格");

    compile = xPath.compile("//car[@year > 1995]/@*|//car[@year > 1995]/price/text()");

    printNodeValue(parse, compile);

}

/***
 * 打印节点值
 * **/
private static void printNodeValue(Document parse, XPathExpression compile) throws XPathExpressionException {
    Object evaluate = compile.evaluate(parse, XPathConstants.NODESET);

    NodeList nodeList = (NodeList) evaluate;
    for (int i = 0; i < nodeList.getLength(); i++) {
        System.out.println(nodeList.item(i).getNodeValue());
    }
}
```


以上代码运行结果为

```java
error:文档根元素 "inventory" 必须匹配 DOCTYPE 根 "null"。
error:文档无效: 找不到语法。
查询颜色是黑色的汽车
blackCar
查询价格小于于19000的汽车
blueCar
查询1999年后所有汽车的属性和价格
2000
14000
2005
24000
```

有的同学可能会注意到前面两句`error:文档根元素 "inventory" 必须匹配 DOCTYPE 根 "null"。  error:文档无效: 找不到语法。`  
这个是因为之前我们配置了`documentBuilderFactory.setValidating(true);`校验文档功能的开启，并且没有提供相应的DTD文件造成的，简单来说，就是开启校验文档功能后，XML文件中要指定DTD语法文档，我们没有指定DTD文件，所以就会报错，关于此处以后会详细说明，有兴趣的同学可以去了解一下”XML验证“。

## MyBatis中的使用 ##

### XPathParser ###

`MyBatis`中提供了`XPathParser`类，此类封装了前面提到了`Xpath`，`Document`，`EntityResolver`等，此类有如下属性  

```java
  /**
   * Document对象
   * */
  private Document document;
  /**
   * 是否开启验证
   * */
  private boolean validation;
  /**
   * 如果开启验证，会首先从http://mybatis.org/dtd/mybatis-3-config.dtd下载dtd文件进行校验
   * 但是实际情况由于网速问题可能会导致下载缓慢，EntityResolver接口负责查找本地的dtd文件
   * */
  private EntityResolver entityResolver;
  /**
   * mybatis-config.xml中 <properties></>标签定义的键位对集合
   * */
  private Properties variables;
  /**
   * 用户查找标记的xpath对象
   * */
  private XPath xpath;
```

上面我们说，`EntityResolver`负责查找本地的`dtd`文件，`XMLMapperEntityResolver` 就是是`MyBatis` 提供的`EntityResolver` 接口的实现类，类图如下


![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/XMLMapperEntityResolver.png)

`EntityResolver`中只有一个方法`resolveEntity()`，一起看下在`XMLMapperEntityResolver`中的实现


```java
	//MyBatis和IBatis中DTD文件的systemId
	private static final String IBATIS_CONFIG_SYSTEM = "ibatis-3-config.dtd";
	private static final String IBATIS_MAPPER_SYSTEM = "ibatis-3-mapper.dtd";
	private static final String MYBATIS_CONFIG_SYSTEM = "mybatis-3-config.dtd";
	private static final String MYBATIS_MAPPER_SYSTEM = "mybatis-3-mapper.dtd";
	
	//DTD文件的具体位置
	private static final String MYBATIS_CONFIG_DTD = "org/apache/ibatis/builder/xml/mybatis-3-config.dtd";
	private static final String MYBATIS_MAPPER_DTD = "org/apache/ibatis/builder/xml/mybatis-3-mapper.dtd";
```


​		
```java
  /**
   * publicId：dtd文件的网络位置，如http://mybatis.org/dtd/mybatis-3-config.dtd
   * 根据对应的systemId（如mybatis-3-mapper.dtd）找到对应的dtd文件
   * 并封装成inputSource文件返回，此类允许SAX应用程序在单个对象中封装有关输入源的信息
   */
  @Override
  public InputSource resolveEntity(String publicId, String systemId) throws SAXException {
    try {
      if (systemId != null) {
        //转换为小写
        String lowerCaseSystemId = systemId.toLowerCase(Locale.ENGLISH);
        //根据不同的systemId找到对应的文件
        if (lowerCaseSystemId.contains(MYBATIS_CONFIG_SYSTEM) || lowerCaseSystemId.contains(IBATIS_CONFIG_SYSTEM)) {
          return getInputSource(MYBATIS_CONFIG_DTD, publicId, systemId);
        } else if (lowerCaseSystemId.contains(MYBATIS_MAPPER_SYSTEM) || lowerCaseSystemId.contains(IBATIS_MAPPER_SYSTEM)) {
          return getInputSource(MYBATIS_MAPPER_DTD, publicId, systemId);
        }
      }
      return null;
    } catch (Exception e) {
      throw new SAXException(e.toString());
    }
  }

  /**
   * 根据路径找到对应的dtd本地文件并返回
   * **/
  private InputSource getInputSource(String path, String publicId, String systemId) {
    InputSource source = null;
    if (path != null) {
      try {
        InputStream in = Resources.getResourceAsStream(path);
        source = new InputSource(in);
        source.setPublicId(publicId);
        source.setSystemId(systemId);        
      } catch (IOException e) {
        // 忽略，找不到就直接返回null
      }
    }
    return source;
  }	
```

介绍完了`XMLMapperEntityResolver`，回到`XPathParser`中，`XPathParser`中有个比较重要的方法`createDocument()`，此方法中封装了我们之前提到的使用`Document `对象和`XPath`解析`XML`文件的代码

```java
private void commonConstructor(boolean validation, Properties variables, EntityResolver entityResolver) {
    this.validation = validation;
    this.entityResolver = entityResolver;
    this.variables = variables;
    XPathFactory factory = XPathFactory.newInstance();
    this.xpath = factory.newXPath();
}
```


```java
  private Document createDocument(InputSource inputSource) {
    // 重要：此方法必须在commonConstructor之后调用
    try {
      DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
      //设置是否验证
      factory.setValidating(validation);

      //提供对XML命名空间的支持
      factory.setNamespaceAware(false);
      //忽略备注
      factory.setIgnoringComments(true);
      //忽略空格
      factory.setIgnoringElementContentWhitespace(false);
      //把CDATA节点转换为text节点，并将其附加到相邻的文本节点（如果有）
      factory.setCoalescing(false);
      //展开实体引用节点
      factory.setExpandEntityReferences(true);

      DocumentBuilder builder = factory.newDocumentBuilder();
      //设置EntityResolver接口对象，避免网络请求dtd文件
      builder.setEntityResolver(entityResolver);
      //如果解析失败则抛出异常
      builder.setErrorHandler(new ErrorHandler() {
        @Override
        public void error(SAXParseException exception) throws SAXException {
          throw exception;
        }

        @Override
        public void fatalError(SAXParseException exception) throws SAXException {
          throw exception;
        }

        @Override
        public void warning(SAXParseException exception) throws SAXException {
        }
      });
      //创建Document对象
      return builder.parse(inputSource);
    } catch (Exception e) {
      throw new BuilderException("Error creating document instance.  Cause: " + e, e);
    }
  }
```

看到上面的代码是不是很熟悉，没错，这就是我们之前所写的示例代码，可以看到，此方法构建了一个`DocumentBuilder`，并且交由此类去解析输入源。

#### 解析字符串 ####

`XPathParser`类中提供了很多`eval*()`方法去解析数据源，原理就是调用`XPath`的`xpath.evaluate()`方法去解析，具体比较简单，这里只分析其中一个比较复杂的方法`evalString()`


```java
  /**
   * 将表达式解析为字符串类型返回，
   * 内部调用PropertyParser.parse()，该方法会处理占位符，默认值等
   * */
  public String evalString(Object root, String expression) {
    String result = (String) evaluate(expression, root, XPathConstants.STRING);
    result = PropertyParser.parse(result, variables);
    return result;
  }
```

可以看到第一句与其它的`eval`方法无异，都是调用`XPath`的接口，第二句则调用了`PropertyParser`的`parse()`方法来解析数据并返回`String`类型的变量。此类会检测是否开启默认值的功能以及修改默认分隔符，默认值以及默认分隔符功能官网说明如下

![image-20201231115344812](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/picgo/image-20201231115344812.png)

`PropertyParser.parse()`方法实现如下

```java
private static final String KEY_PREFIX = "org.apache.ibatis.parsing.PropertyParser.";

//是否开启默认值功能的配置项
public static final String KEY_ENABLE_DEFAULT_VALUE = KEY_PREFIX + "enable-default-value";

//默认值和占位符之间的分隔符配置项
public static final String KEY_DEFAULT_VALUE_SEPARATOR = KEY_PREFIX + "default-value-separator";

//是否开启默认值
private static final String ENABLE_DEFAULT_VALUE = "false";

//分隔符默认值	
private static final String DEFAULT_VALUE_SEPARATOR = ":";

public static String parse(String string, Properties variables) {
    VariableTokenHandler handler = new VariableTokenHandler(variables);
    GenericTokenParser parser = new GenericTokenParser("${", "}", handler);
    return parser.parse(string);
}
```


可以看到，此方法创建了一个GenericTokenParser解析器，并委托给`GenericTokenParser.parse()`进行解析，此方法逻辑并不复杂，只是一次查找开始和结束标记，并把两个标记中间的文本值交给`TokenHandler`处理，最后将结果返回，实现如下


```java
  public String parse(String text) {
    //如果为空直接返回
    if (text == null || text.isEmpty()) {
      return "";
    }
    char[] src = text.toCharArray();
    int offset = 0;
    // 查找开始标记的位置
    int start = text.indexOf(openToken, offset);
    //如果不存在则直接返回
    if (start == -1) {
      return text;
    }
    //builder用来保存解析后的字符串
    final StringBuilder builder = new StringBuilder();
    //expression用来保存占位符内的公式
    StringBuilder expression = null;
    while (start > -1) {
      /*
      * 如果开始字符前面是转义字符“\”，则去掉转义字符
      * 比如\${abc，会拼接成${abc
      * */
      if (start > 0 && src[start - 1] == '\\') {
        builder.append(src, offset, start - offset - 1).append(openToken);
        offset = start + openToken.length();
      } else {
        // 重置expression对象
        if (expression == null) {
          expression = new StringBuilder();
        } else {
          expression.setLength(0);
        }
        /*
         * 首先拼接offset到开始字符之间的普通字符串（此时的offset为0）
         * 如test${id_var}，则先将test拼接到结果中
         * */
        builder.append(src, offset, start - offset);
        //将offset更改为开始字符后面的的位置
        offset = start + openToken.length();
        //从offset的位置开始查找结束标记的位置
        int end = text.indexOf(closeToken, offset);
        //如果找到了结束标记
        while (end > -1) {
          //处理结束标记的转移字符
          if (end > offset && src[end - 1] == '\\') {
            expression.append(src, offset, end - offset - 1).append(closeToken);
            offset = end + closeToken.length();
            end = text.indexOf(closeToken, offset);
          } else {
            /*
             * 将开始标记和结束标记之间的字符拼接到表达式中
             * 如${abc}，这里会拼接abc
             * */
            expression.append(src, offset, end - offset);
            //重新计算offset
            offset = end + closeToken.length();
            break;
          }
        }
        if (end == -1) {
          // 如果找不到关闭字符，则将开始字符后面的文本拼接起来
          builder.append(src, start, src.length - start);
          offset = src.length;
        } else {
          //将占位符里面的表达式文本交给TokenHandler处理并拼接到结果中
          builder.append(handler.handleToken(expression.toString()));
          offset = end + closeToken.length();
        }
      }
      //再次开始查找
      start = text.indexOf(openToken, offset);
    }
    if (offset < src.length) {
      builder.append(src, offset, src.length - offset);
    }
    return builder.toString();
  }
```

占位符文本值的解析委托给`TokenHandler.handleToken()`来实现，`TokenHandler`有四个接口，如下

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/TokenHandler%E7%B1%BB%E5%9B%BE.png)

由`PropertyParser`可以得知，此处调用的是`VariableTokenHandler`，这是一个`PropertyParser`的内部类，实现如下

```java
    /**
     * properties标签中配置的键值对
     * */
    private final Properties variables;
    /**
     * 是否开启默认值功能
     * */
    private final boolean enableDefaultValue;
    /**
     * 默认分隔符
     * */
    private final String defaultValueSeparator;
```


​	

```java
    public String handleToken(String content) {
      if (variables != null) {
        String key = content;
        //如果开启了默认值功能
        if (enableDefaultValue) {
          //查找是否有分隔符
          final int separatorIndex = content.indexOf(defaultValueSeparator);
          String defaultValue = null;
          if (separatorIndex >= 0) {
            /*
            * 如果有分隔符，则分割key和后面的默认值，如  ${username:abc}
            * 这里的key就是username，默认值就是abc
            * */
            key = content.substring(0, separatorIndex);
            defaultValue = content.substring(separatorIndex + defaultValueSeparator.length());
          }
          if (defaultValue != null) {
            //如果找到就返回，找不到就返回默认值
            return variables.getProperty(key, defaultValue);
          }
        }
        //如果properties中配置的键值对中存在该key，则从中获取
        if (variables.containsKey(key)) {
          return variables.getProperty(key);
        }
      }
      //如果properties没有配置键值对的话，这里就直接返回
      return "${" + content + "}";
    }
```

在上述代码中，如果我们有一个配置项为`${username:root}`，则首先会去配置文件中查找`username`的`value`，如果查找不到，则直接返回`root`。  
需要注意的是，`GenericTokenParser`还会用在之后的SQL语句解析，很明显，此类只是查找开始和结束标记，具体的处理方式会根据`TokenHandler`的不同而不同，这里也是策略模式的一种使用方式。  


#### 解析Node节点 ####

回到`XPathParser`类中，该类有一个`evalNode()`方法，观察方法名可以看出这是个解析`node`节点的方法，此方法返回`XNode`对象，此类是`MyBatis`对`org.w3c.dom.Node`进行的一些简单封装，该类主要字段如下

```java
  /**
   * 节点对象
   * */
  private Node node;
  /**
   * 节点的名称
   * */
  private String name;
  /**
   * 节点内容
   * */
  private String body;
  /**
   * 节点的属性集合
   * */
  private Properties attributes;
  /**
   * properties标签中配置的键值对
   * */
  private Properties variables;
  /**
   * xpathParser对象，标记该xnode对象由哪个解析器生成
   * */
  private XPathParser xpathParser;

  public XNode(XPathParser xpathParser, Node node, Properties variables) {
    this.xpathParser = xpathParser;
    this.node = node;
    this.name = node.getNodeName();
    this.variables = variables;
    //解析该节点的属性
    this.attributes = parseAttributes(node);
    //解析该结点的内容
    this.body = parseBody(node);
  }
```
可以看到，该类初始化时，调用了`parseAttributes()`和`parseBody()`方法进行对节点内容和节点属性的解析，两个方法实现如下

```java
  private Properties parseAttributes(Node n) {
    Properties attributes = new Properties();
    //调用org.w3c.dom.Node.getAttributes()方法获取该节点的属性
    NamedNodeMap attributeNodes = n.getAttributes();
    //如果不为null，则调用PropertyParser对象一个个处理占位符后放入attributes集合中
    if (attributeNodes != null) {
      for (int i = 0; i < attributeNodes.getLength(); i++) {
        Node attribute = attributeNodes.item(i);
        String value = PropertyParser.parse(attribute.getNodeValue(), variables);
        attributes.put(attribute.getNodeName(), value);
      }
    }
    return attributes;
  }

  private String parseBody(Node node) {
    String data = getBodyData(node);
    //如果不是text节点，则获取子节点的内容
    if (data == null) {
      NodeList children = node.getChildNodes();
      for (int i = 0; i < children.getLength(); i++) {
        Node child = children.item(i);
        data = getBodyData(child);
        if (data != null) {
          break;
        }
      }
    }
    return data;
  }

  private String getBodyData(Node child) {
    //只处理text节点或CDATA节点
    if (child.getNodeType() == Node.CDATA_SECTION_NODE
        || child.getNodeType() == Node.TEXT_NODE) {
      //调用org.w3c.dom.CharacterData.getData()方法获取节点内容
      String data = ((CharacterData) child).getData();
      //调用PropertyParser处理占位符
      data = PropertyParser.parse(data, variables);
      return data;
    }
    //如果不是text节点则直接返回null
    return null;
  }
```

`XNode`中提供了多个`get*()`方法来获取值，而这些方法基本也都是操作上述的属性来获取相应的值，逻辑比较简单，比如

```java
  public Long getLongAttribute(String name, Long def) {
    String value = attributes.getProperty(name);
    if (value == null) {
      return def;
    } else {
      return Long.parseLong(value);
    }
  }
```

当然，也可以通过`eval*()`系列的方法来获取想要的值，需要注意的是，这里的`eval*()`对应的上下文是`XNode.Node`，也就是在当前节点寻找符合条件的值。