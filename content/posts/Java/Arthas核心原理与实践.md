---
title: Arthas核心原理与实践
tags:
  - Java
date: 2021-11-15
categories:
 - Java
---

# Arthas核心原理与实践

## 问题由来

​	来自某次线上问题：大仓收货入库时需要调用品控系统接口更新入库单状态，该接口一直无法正常响应数据，导致商品无法及时入库，延误了后续的流程。由于这个接口没有详细的请求过程日志，无法查看核心逻辑的执行流程，幸运的是，通过`review`代码的方式成功的定位到了问题并及时修复。但是事后却引起了深思：这次解决问题靠的是幸运，但是下次就不一定了，在业务比重越来越高的今天，如果日志不能有效的提供帮助，有没有办法能够快速的排查并定位线上问题？

## 技术调研

​	经过了解，发现了多个排查`Java`应用问题的工具，最终选择了两种进行比较。分别是[btrace](https://github.com/btraceio/btrace)和[Arthas](https://github.com/alibaba/arthas)。

|            | `Arthas`                           | `btrace`                                 |
| ---------- | ---------------------------------- | ---------------------------------------- |
| 文档资料   | 官方文档比较丰富，网上资料多       | 官方文档内容一般，网上资料较多           |
| 社区活跃度 | 活跃                               | 一般                                     |
| 使用方法   | 自带脚本库，开箱即用，无需编写代码 | 定义了一套注解，需要使用注解自行编写脚本 |
| 拓展程度   | 可以使用`ognl`表达式拓展           | 可以编写`Java`程序拓展                   |
| 自定义逻辑 | 复杂                               | 简单                                     |
| 可视化界面 | 自带`Web Console`                  | 需要自行实现                             |

​	对比后发现，`btrace`在每次排查问题时，都需要手动编写脚本，在启动时指定脚本位置。而`Arthas`提供了一整套开箱即用的工具便于快速的定位和分析问题。手动编写脚本带来的优势是可以自定义逻辑，便于拓展，而劣势就是大大降低了对于突发事件处理的响应速度和灵活性。所以最终还是选择了`Arthas`。

## 基本介绍

`Arthas`能够帮助我们解决很多问题的诊断与定位，比如下面的这几个场景

1. 提交了代码，但是一直不生效，没办法确定线上运行的是不是提交后的代码。
2. 线上接口一直报错，但是没有加日志，无法通过入参和出参进行判断。
3. 超时问题特别棘手，不知道从哪里开始排查，没有头绪。

这些都是日常开发中非常容易遇到的，而`Arthas`就是为了解决这些问题出现的

## 核心原理

​	`Arthas`的大致原理：启动后，会分别启动服务端和客户端，客户端使用`telnet`连接到服务端，服务端负责接收客户端的命令动态修改被监测类的字节码，并将响应结果发送到客户端。

![Arthas 整体逻辑-正式](/images/oss/picGo/Arthas%20%E6%95%B4%E4%BD%93%E9%80%BB%E8%BE%91-%E6%AD%A3%E5%BC%8F.png)

`Arthas`的工作过程一共分为7个步骤

1. 启动`Arthas-boot.jar`，进行主程序启动前的准备工作。如：检查可用的`Http`和`Telnet`端口，下载`Arthas`程序包等。
2. 启动`arthas-core.jar`，解析启动命令
3. 运行`arthas-agent.jar`，通过`Attach API`连接到应用程序。
4. 启动服务器端，监听客户端命令
5. 启动客户端，将输入的命令传输到服务端
6. 根据命令增强响应的字节码
7. 构造响应结果向客户端输出

以下是每个步骤的具体内容

 Attach API

从`jdk 1.5`开始，`java`推出了`Instrumentation API` 。简单来说，它可以让你单独开发一套独立于应用程序的代理程序（这个代理程序通过`Instrumentation API`实现），用来实现监测或辅助应用程序运行（有点类似于`Spring AOP`）。但是在`jdk 1.5`版本的时候。使用起来很不方便，必须要在启动应用程序时指定`agent`（也就是代理程序），比如下面的命令

```Shell
##agent.jar会在应用程序的main函数启动之前执行



$ java -jar xxxx.jar -javaagent agent.jar
```

所以在`jdk 1.6`版本中，增强了`Instrumentation API`，并允许应用程序先执行，代理程序通过`attach`的方式接入，这样就极大的提升了灵活度和想象空间。`Arthas`就是通过`attach`的方式接入应用程序，并使用`Instrumentation API`完成监控动作的。如下

```Java
//列出所有的java虚拟机列表

List<VirtualMachineDescriptor> list = VirtualMachine.list();



//attach到目标进行上

VirtualMachine attach = VirtualMachine.attach(pid);



//装载代理程序

attach.loadAgent("agent.jar");
```

然后在代理程序中的`agentmain`方法中修改类的字节码

```Java
public static void agentmain(String agentArgs, Instrumentation inst) {

    inst.addTransformer(new ClassFileTransformer() {

         @Override

         public byte[] transform(ClassLoader loader, String className,

                                                Class<?> clazz,

                                           ProtectionDomain protectionDomain,

                    byte[] byteCode) throws IllegalClassFormatException {

                            //这里就可以直接修改类字节码返回，比如加一些监控代码~

                            return byteCode;

                  }

             }

    );

}
```

### 启动：准备

了解`attach`和`agent`的方式之后，下面开始分析`Arthas`的代码

首先根据启动命令`java -jar arthas-boot.jar`得知程序的入口是`arthas-boot.jar`，查看此`jar`包的`pom`文件可以看到启动类是`com.taobao.arthas.boot.Bootstrap`

![image (1)](/images/oss/picGo/image%20(1).png)

在`com.taobao.arthas.boot.Bootstrap.main()`方法中，主要做以下几步

1. 通过`Alibaba`开源的`cli`解析参数

2. 设置下载的镜像为`aliyun`

3. 检查`jdk`版本

4. 检查`telnet`和`http`端口

5. 选择进程`pid`，如果启动命令中没有指定`pid`，则会运行`jps -l`命令列出本机上所有运行的`java`进程，并等待客户端输入（省略部分代码）

   ```java
       pid = ProcessUtils.select(bootstrap.isVerbose(), telnetPortPid, bootstrap.getSelect());
   
   
   
   // ProcessUtils.select()
   
   public static long select()  {
   
         //listProcessByJps方法会使用jps -l列出所有的进程，map的key为进程pid，value为进程名
   
         Map<Long, String> processMap = listProcessByJps(v);
   
   
   
         // 打印找到的进程集合
   
         int count = 1;
   
         for (String process : processMap.values()) {
   
             if (count == 1) {
   
                 System.out.println("* [" + count + "]: " + process);
   
             } else {
   
                 System.out.println("  [" + count + "]: " + process);
   
             }
   
             count++;
   
         }
   
   
   
         // 等待客户端输入
   
         String line = new Scanner(System.in).nextLine();
   
   
   
         int choice = new Scanner(line).nextInt();
   
   
   
         Iterator<Long> idIter = processMap.keySet().iterator();
   
         for (int i = 1; i <= choice; ++i) {
   
             if (i == choice) {
   
                 return idIter.next();
   
             }
   
             idIter.next();
   
         }
   
   
   
         return -1;
   
     }
   ```

6. 查看`arthas`程序包是否存在，如果不存在则下载

7. 拼接启动命令，启动`arthas-core.jar`（省略部分代码）

   ```java
   //这里主要是为了拼接启动命令
   
   List<String> attachArgs = new ArrayList<String>();
   
   attachArgs.add("-jar");
   
   attachArgs.add(new File(arthasHomeDir, "arthas-core.jar").getAbsolutePath());
   
   attachArgs.add("-pid");
   
   attachArgs.add("" + pid);
   
   attachArgs.add("-core");
   
   attachArgs.add(new File(arthasHomeDir, "arthas-core.jar").getAbsolutePath());
   
   attachArgs.add("-agent");
   
   attachArgs.add(new File(arthasHomeDir, "arthas-agent.jar").getAbsolutePath());
   
   
   
   ProcessUtils.startArthasCore(pid, attachArgs);
   
   
   
   //ProcessUtils.startArthasCore()
   
   public static void startArthasCore(long targetPid, List<String> attachArgs) {
   
       //最终command会被拼接成这样：java -jar arthas-core.jar -pid xxx -core arthas-core.jar -agent arthas-agent.jar
   
       ProcessBuilder pb = new ProcessBuilder(command);
   
       final Process proc = pb.start();
   
       Thread redirectStdout = new Thread(new Runnable() {
   
           @Override
   
           public void run() {
   
               InputStream inputStream = proc.getInputStream();
   
               try {
   
                   //将启动的日志展示到控制台
   
                   IOUtils.copy(inputStream, System.out);
   
               } catch (IOException e) {
   
                   IOUtils.close(inputStream);
   
               }
   
       
   
           }
   
       });
   
   }
   ```

### 启动：arthas-core

从上文可以得知，最终的启动命令会被拼接成这样：`java -jar arthas-core.jar -pid xxx -core arthas-core.jar -agent arthas-agent.jar`。`arthas-core.jar`的`pom`文件中指定了`main-class`为`com.taobao.arthas.core.Arthas`。

该类逻辑比较简单

1、通过`attach API`连接目标进程

2、解析各种启动参数然后加载`agent.jar`

```Java
public class Arthas {



    private Arthas(String[] args) throws Exception {

        //parse()方法用来解析各种参数

        attachAgent(parse(args));

    }



   

    private void attachAgent(Configure configure) throws Exception {

        VirtualMachineDescriptor virtualMachineDescriptor = null;

        for (VirtualMachineDescriptor descriptor : VirtualMachine.list()) {

            String pid = descriptor.id();

            if (pid.equals(configure.getJavaPid())) {

                virtualMachineDescriptor = descriptor;

                break;

            }

        }

        VirtualMachine virtualMachine = null;

        if (null == virtualMachineDescriptor) { 

            //连接到目标进程中~

            virtualMachine = VirtualMachine.attach(configure.getJavaPid());

        } else {

            virtualMachine = VirtualMachine.attach(virtualMachineDescriptor);

        }

        String arthasAgentPath = configure.getArthasAgent();

        //加载agent，这句是不是也很熟悉

        virtualMachine.loadAgent(arthasAgentPath);

        

    }

    

    public static void main(String[] args) {

        new Arthas(args);

    }

}
```

### 加载：arthas-agent

上面说到`arthas-core.jar`会注入`arthas-agent.jar`。根据前面介绍过的`agent`可以得知，执行的应该是`arthas-agent`的`premain()`方法。

`arthas-agent`的`pom`文件指定了该`jar`的`premain-class`

![image-20211220175636850](/images/oss/picGo/image-20211220175636850.png)

所以就可以直接看`com.taobao.arthas.agent334.AgentBootstrap.premain()`了，这里使用了`jdk 1.5`的方式直接指定的`agent`，所以是`premain`方法。如果是`1.6`的`attach`方式，那就是`agentmain`方法。

```Java
public static void premain(String args, Instrumentation inst) {

    main(args, inst);

}





private static synchronized void main(String args, final Instrumentation inst) {

    // 尝试判断arthas是否已在运行，如果已经运行就不再加载

    Class.forName("java.arthas.SpyAPI"); // 加载不到会抛异常 

    if (SpyAPI.isInited()) {

        ps.println("Arthas server already stared, skip attach.");

        ps.flush();

        return;

    }

    

    //省略部分代码。。。

    

    Thread bindingThread = new Thread() {

        @Override

        public void run() {

            try {

                //注意看这里，bind方法会启动server端并绑定端口号

                bind(inst, agentLoader, agentArgs);

            } catch (Throwable throwable) {

                throwable.printStackTrace(ps);

            }

        }

    };

    

    bindingThread.setName("arthas-binding-thread");

    bindingThread.start();

    bindingThread.join();

}
```

进入`bind()`方法中，这里调用了`arthas-core.jar`中的`com.taobao.arthas.core.server.ArthasBootstrap.getInstance()`

```Java
private static final String ARTHAS_BOOTSTRAP = "com.taobao.arthas.core.server.ArthasBootstrap";

private static final String GET_INSTANCE = "getInstance";



private static void bind(Instrumentation inst, ClassLoader agentLoader, String args)  {

    //agentLoader已经提前加载了arthas-core.jar

    Class bootstrapClass = agentLoader.loadClass(ARTHAS_BOOTSTRAP);

    Object bootstrap = bootstrapClass.getMethod(GET_INSTANCE, Instrumentation.class, String.class).invoke(null, inst, args);

    //省略部分逻辑 

}
```

`com.taobao.arthas.core.server.ArthasBootstrap.getInstance()`方法比较简单，就是实例化了`com.taobao.arthas.core.server.ArthasBootstrap`，该类的构造方法中会调用`bind()`开启服务端，同时初始化所有的`command`。

```Java
private void bind(Configure configure) {



    //1、初始化所有的command

    BuiltinCommandPack builtinCommands = new BuiltinCommandPack();

    //2、找到一个可用的Telnet端口来开启服务器

    //3、找到一个可用的Http端口来开启服务器

    //4、注册Telnet服务器端

    shellServer.registerTermServer(new HttpTelnetTermServer());

    //5、注册Http服务器端

    shellServer.registerTermServer(new HttpTermServer());

    

    //  调用刚才注册的两个服务器端的listen()方法

    shellServer.listen(new BindHandler(isBindRef));

}
```

在`BuiltinCommandPack`的构造方法中，会调用自身的`initCommands()`来初始化所有`command`

```Java
private void initCommands() {

    commandClassList.add(JadCommand.class);

    commandClassList.add(TraceCommand.class);

    commandClassList.add(WatchCommand.class);

    //省略其他。。。

}

//XXXCommand继承了AnnotatedCommand接口，并实现process()方法处理对应命令的相关逻辑

//这里声明对应的命令

@Name("jad")

public class JadCommand extends AnnotatedCommand {

   

    @Override

    public void process(CommandProcess process) {

       // process()的逻辑放在后面分析

    }

}
```

接下来继续说服务器启动逻辑。`shellServer.listen()`会遍历刚才注册的两个服务器端，并分别执行`listen()`方法。这里选择`HttpTelnetTermServer.listen()`（实现原理都是一样的）

```Java
public TermServer listen(Handler<Future<TermServer>> listenHandler) {

    // 使用netty启动服务器端

    bootstrap = new NettyHttpTelnetTtyBootstrap(workerGroup, httpSessionManager).setHost(hostIp).setPort(port);

    bootstrap.start(new Consumer<TtyConnection>() {

            @Override

            public void accept(final TtyConnection conn) {

                //这里的实现是TermServerTermHandler

                termHandler.handle(new TermImpl());

            }

     }).get(connectionTimeout, TimeUnit.MILLISECONDS);

}
```

可以看到，`listen()`方法会使用`netty`启动服务器端，并且为每个连接的客户端绑定回调方法`termHandler.handle()`。

`termHandler.handle()`方法中调用`com.taobao.arthas.core.shell.impl.ShellServerImpl#handleTerm()`实现`banner`的打印（就是刚连接上来的那个`ARTHAS`彩色英文）以及客户端命令的监听等，如下

```Java
public void handleTerm() {

    ShellImpl session = createShell(term);

    //welcomeMessage就是刚连接上来展示的banner

    session.setWelcome(welcomeMessage);

    

    //init()方法会输出上面赋值的welcomeMessage

    session.init();

    

    //开始读取客户端命令，并创建回调

    session.readline(); 

}



//读取客户端输入，并为每个输入命令绑定ShellLineHandler处理

public void readline() {

    term.readline(prompt, new ShellLineHandler(this), new CommandManagerCompletionHandler(commandManager));

}
```

到此为止，服务器启动就分析完了，接下来用一张图总结

![image-20211220175719701](/images/oss/picGo/image-20211220175719701.png)

### 增强

上面说到，服务器端会读取每个输入命令并为其绑定`ShellLineHandler`处理，`ShellLineHandler.handle()`会为输入的命令创建一个`Job`对象并执行对应逻辑

```Java
public void handle(String line) {

    List<CliToken> tokens = CliTokens.tokenize(line);

    CliToken first = TokenUtils.findFirstTextToken(tokens);

    if (first == null) {

        shell.readline();

        return;

    }



    //如果是exit、logout、q、quit等简单命令，就不需要封装job了，直接执行即可

    String name = first.value();

    if (name.equals("exit") || name.equals("logout") || name.equals("q") || name.equals("quit")) {

        handleExit();

        return;

    } else if (name.equals("jobs")) {

        handleJobs();

        return;

    } else if (name.equals("fg")) {

        handleForeground(tokens);

        return;

    } else if (name.equals("bg")) {

        handleBackground(tokens);

        return;

    } else if (name.equals("kill")) {

        handleKill(tokens);

        return;

    }



    //为每个命令创建一个job对象并执行

    Job job = createJob(tokens);

    if (job != null) {

        job.run();

    }

}
```

`createJob()`最终会调用到`com.taobao.arthas.core.shell.system.impl.JobControllerImpl#createJob()`中，该方法会通过输入的命令获取前面提到过的`Command`，并将`Command`封装为`process`，最终包装成`job`返回。

```Java
public Job createJob() {

    int jobId = idGenerator.incrementAndGet();

    //根据输入命令创建process

    Process process = createProcess();

    process.setJobId(jobId);

    //将process封装为job返回

    JobImpl job = new JobImpl(process);

    jobs.put(jobId, job);

    return job;

}







private Process createProcess() {

     ListIterator<CliToken> tokens = line.listIterator();

        while (tokens.hasNext()) {

            CliToken token = tokens.next();

            if (token.isText()) {

                //根据客户端输入获取command，并封装为process返回

                Command command = commandManager.getCommand(token.value());

                if (command != null) {

                    return createCommandProcess();

                } else {

                    throw new IllegalArgumentException("。。。");

                }

            }

        }

}
```

获取到`job`之后，接下来就是执行`job.run()`方法，该方法最终会调用到`com.taobao.arthas.core.shell.system.impl.ProcessImpl#run()`中

```Java
public synchronized void run() {

    Runnable task = new CommandProcessTask(process);

    ArthasBootstrap.getInstance().execute(task);

}
```

可以看到，这里会取出`job`中封装的`process`，并封装为`Runnable`执行。

在`Runnable.run()`中，会经过一系列的调用链，拿`watch`命令举例，这里最终调用到`WatchCommand.process()`方法，但是`WatchCommand`并没有实现此方法，所以由其父类—`EnhancerCommand`提供默认实现

```Java
//Runnable.run()

public void run() {

    handler.handle(process);

}



//ProcessHandler.handle()

public void handle(CommandProcess process) {

    process(process);

}



//AnnotatedCommandImpl.process()

private void process(CommandProcess process) {

    AnnotatedCommand instance;

    //watchcommand

    instance = clazz.newInstance();

    //这里调用的是 EnhancerCommand.process()

    instance.process(process);

}
```

`EnhancerCommand.process()`方法首先会获取`AdviceListener`，并调用其内部方法`enhance()`完成对类的最终增强。`AdviceListener`类似于`AOP`的切点，其内部定义了一系列监测的节点，如下

```Java
public interface AdviceListener {

    void before() throws Throwable;



    void afterReturning() throws Throwable;



    void afterThrowing() throws Throwable;

}
```

定义好切点后，各子类只需要实现对应的方法，就可以将监测的代码修改到应用程序的字节码中了，在方便的同时也极具拓展性。

回到之前的思路，前面说`EnhancerCommand.process()`方法会调用其内部方法`enhance()`完成对类的增强，下面简单分析`enhance()`方法（已省略部分代码）

```Java
protected void enhance(CommandProcess process) {

    Enhancer enhancer = new Enhancer();

    //这里会传入Instrumentation

    effect = enhancer.enhance(inst);

}





public synchronized EnhancerAffect enhance(final Instrumentation inst)  {

    //首先将自身加入到ClassFileTransformer中

    ArthasBootstrap.getInstance().getTransformerManager().addTransformer(this, isTracing);

    

    //增强

    inst.retransformClasses(clazz);

}
```

由于`Enhancer`实现了`ClassFileTransformer`接口，并首先将自身添加进了`ClassFileTransformer`中，所以在调用`Instrumentation.retransformClasses()`方法后，会由其自身的`transform()`完成最终增强，方法内部使用 `alibaba`的`bytekit`实现字节码的修改

```Java
public byte[] transform() {

    //SpyInterceptor*.class中会在需要增强的方法中插入一个钩子，用来在方法入口、出口、异常时执行相应方法

    interceptorProcessors.addAll(defaultInterceptorClassParser.parse(SpyInterceptor1.class));

    interceptorProcessors.addAll(defaultInterceptorClassParser.parse(SpyInterceptor2.class));

    interceptorProcessors.addAll(defaultInterceptorClassParser.parse(SpyInterceptor3.class));

    //省略部分代码

}





public static class SpyInterceptor1 {



    @AtEnter(inline = true)

    public static void atEnter() {

        //在方法入口会执行SpyAPI.atEnter方法，并携带方法、类、参数等信息

        SpyAPI.atEnter(clazz, methodInfo, target, args);

    }

}
```

`SpyAPI.atEnter()`方法由`SpyImpl.atEnter()`方法由提供实现，使用前面提到的`AdviceListener`完成各种命令的功能

用一张图来总结这部分的流程。

![增强-正式](/images/oss/picGo/%E5%A2%9E%E5%BC%BA-%E6%AD%A3%E5%BC%8F.png)

#### watch

`watch`命令可以监测方法调用时的入参、返回值等

![image-20211220175809701](/images/oss/picGo/image-20211220175809701.png)

主要代码逻辑在`WatchAdviceListener`中

```Java
class WatchAdviceListener extends AdviceListenerAdapter {

    @Override

    public void before() {

        // 开始计算本次方法调用耗时

        threadLocalWatch.start();

        if (command.isBefore()) {

            watching();

        }

    }



    @Override

    public void afterReturning() {

        Advice advice = Advice.newForAfterRetuning();

        if (command.isSuccess()) {

            watching(advice);

        }



        finishing(advice);

    }



    private void watching(Advice advice) {

        // 本次调用的耗时

        double cost = threadLocalWatch.costInMillis();

        //根据命令行传入的ognl表达式参数获取实际值，如方法参数、方法返回值

        Object value = getExpressionResult(command.getExpress(), advice, cost);

        //组装响应结果返回客户端

        WatchModel model = new WatchModel();

        model.setTs(new Date());

        model.setCost(cost);

        model.setValue(value);

        model.setExpand(command.getExpand());

        model.setSizeLimit(command.getSizeLimit());

        model.setClassName(advice.getClazz().getName());

        model.setMethodName(advice.getMethod().getName());

    }

}
```

#### trace

`trace`命令可以打印出方法执行的整体调用链路

![image-20211220175833420](/images/oss/picGo/image-20211220175833420.png)

`TraceAdviceListener`没有实现`before()`方法，由父类`AbstractTraceAdviceListener`提供实现

```Java
public class AbstractTraceAdviceListener extends AdviceListenerAdapter {

    public void before() {

        TraceEntity traceEntity = threadLocalTraceEntity(loader);

        //使用TraceEntity.begin()方法标记整条调用链路的开始

        traceEntity.tree.begin(clazz.getName(), method.getName(), -1, false);

        traceEntity.deep++;

        // 开始计算本次方法调用耗时

        threadLocalWatch.start();

    }

}
```

`TraceEntity.tree`中维护了一个树形结构，用来记录每个方法的调用情况。每次使用`trace`命令时，都会新建一个`TraceTree`对象。

```Java
public class TraceTree {

    private TraceNode root;

    private TraceNode current;

}



public abstract class TraceNode {

    protected TraceNode parent;

    protected List<TraceNode> children;

}
```

而每次的方法内部调用，都会调用到`TraceAdviceListener.invokeBeforeTracing()`，该方法会将调用的`method`信息抽象为`TraceNode`节点对象并关联到前面`TraceTree`对象中。

```Java
public void begin() {

    TraceNode child = findChild(current, className, methodName, lineNumber);

    if (child == null) {

        //MethodNode做为TraceNode的实现类，其begin()会记录自身方法的执行时间，这样就可以实现打印每个内部方法调用的耗时

        child = new MethodNode(className, methodName, lineNumber, isInvoking);

        current.addChild(child);

    }

    child.begin();

}
```

### 客户端连接：arthas-client

客户端的逻辑就是连上服务端，开始传输命令并接收服务器的响应结果

`arthas-client.jar`中的`pom`文件指定了`main-class`为`TeletConsole`

![image-20211220175857206](/images/oss/picGo/image-20211220175857206.png)

`TeletConsole.main()`中使用`ConsoleReader`和`TelnetClient`分别处理本地和远程的输入和输出

```Java
public static void main(String[] args) throws Exception {

        int status = process();

        System.exit(status);
}


public static int process() {
    //使用ConsoleReader处理控制台输入和输出

    final ConsoleReader consoleReader = new ConsoleReader(System.in, System.out);

    final TelnetClient telnet = new TelnetClient();

    //该方法内部会使用java.net.Socket.connect()进行远程连接

    telnet.connect(telnetConsole.getTargetIp(), telnetConsole.getPort());

    //监测本地和远程的输入输出

    IOUtil.readWrite(telnet.getInputStream(), telnet.getOutputStream(), consoleReader.getInput(), consoleReader.getOutput());

}
```

## 实际案例

### 实践一：通过Spring容器获取配置项

在实际的开发中，总会有这样的问题：想要获取某个配置文件的值，由于配置存储在`spring`容器中，除非通过`Spring Actuator`，否则根本无法获取到。而使用`Arthas`就可以很轻松的做到这一点

首先通过`ognl`表达式来获取`spring`上下文，其原理就是获取类中引用的`ApplicationContext`变量，这里有几种思路

​    第一种，也是最简单的一种，在程序中直接声明此变量，并通过`ApplicationContextAware`接口赋值

```Java
@Component("springUtils")

public class SpringUtils implements ApplicationContextAware {



    private ApplicationContext context;



    @Override

    public void setApplicationContext(ApplicationContext applicationContext) {

        this.context = applicationContext;

    }



}
```

通过`ognl`直接获取

![image-20211220175912235](/images/oss/picGo/image-20211220175912235.png)

可以看到，容器中的内容还是挺多的，比如`environment`、`beanFactory`等。

第二种，如果程序中没有此实现，但是引入了`dubbo`，`dubbo`中有个类也引入了此变量

```Java
public class SpringExtensionFactory implements ExtensionFactory {
    private static final Set<ApplicationContext> contexts = new ConcurrentHashSet<ApplicationContext>();
}
```

那么就可以通过如下方式获取

```Shell
[arthas@61462]$ ognl '@com.alibaba.dubbo.config.spring.extension.SpringExtensionFactory@contexts.iterator.next'
```

第三种，如果也没有引用`dubbo`的话，那就要想办法获取到一个`spring`自己的对象，再通过`spring`的对象获取`ApplicationContext`。

这里选择通过`tt`命令来监测`spring`处理请求的入口，再通过入口类拿到`ApplicationContext`，怎么获得入口呢？使用上面提过的`stack`命令，随便监测一个`controller`的方法，然后请求一次，如下

```Shell
[arthas@60712]$ stack com.mryx.xx.web.controller.xx.xxxController methodName
```

发起一次请求后，响应结果如下，就可以很方便的找到请求入口了

![image-20211220175934027](/images/oss/picGo/image-20211220175934027.png)

下一步就是使用`tt`命令监测这个方法，然后再重新发起一次请求

```Shell
[arthas@60712]$ tt -t org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter handleInternal
```

![image-20211220175944172](/images/oss/picGo/image-20211220175944172.png)

监测到这个方法后，就可以调用`RequestMappingHandlerAdapter.getApplicationContext()`方法获取`applicationContext`对象

```Shell
# 注意 这里-i 1000对应的是上面监测到请求的index

[arthas@60712]$ tt -i 1000 -w 'target.getApplicationContext()'
```

1. 到目前为止，就已经通过三种方式获取到了`spring`的上下文对象，那么获取到这个对象有啥用呢？当然是非常有用了，比如前面说的，获取某个配置项

```Shell
[arthas@61462]$ ognl '#context=@com.mryx.qms.util.SpringUtils@context, #context.getEnvironment().getProperty("server.port")'
```

![image-20211220175954723](/images/oss/picGo/image-20211220175954723.png)

比如执行某个`service`的方法

```Shell
[arthas@61462]$ ognl '#context=@com.mryx.qms.util.SpringUtils@context, #context.getBean("sysUserService").deleteUser(1)'
```

![image-20211220180006097](/images/oss/picGo/image-20211220180006097.png)

比如。。。。

### 实践二：Spring启动缓慢的排查记录

首先描述下背景，公司的项目，使用`SpringBoot-1.5.13.RELEASE` 版本，该项目启动异常缓慢，达到了`251`秒

![image-20211220180016176](/images/oss/picGo/image-20211220180016176.png)

对于`Arthas`来说，排查这种启动慢的问题是比较麻烦的，因为`Arthas`只有在应用启动后才能连上去，但是启动之后又不能排查启动慢的问题。有一种套圈的感觉，所以这里用了个“聪明”方法，在程序的`main`函数中加入`sleep`

```Java
public static void main(String[] args) {

    try {

        Thread.sleep(30 * 1000);

    } catch (InterruptedException e) {

        e.printStackTrace();

    }

    SpringApplication.run(Application.class, args);

    logger.info("SpringBoot Start Success");

}
```

接下来启动程序后使用`Arthas`开始分析

1. 首先确认当前`classLoader`的`hash`值

```Shell
[arthas@52461]$ classloader -t

    +-BootstrapClassLoader

    +-javax.management.remote.rmi.NoCallStackClassLoader@2a2d45ba

    +-javax.management.remote.rmi.NoCallStackClassLoader@68be2bc2

    +-sun.misc.Launcher$ExtClassLoader@710726a3

      +-com.taobao.arthas.agent.ArthasClassloader@7753b3a8

      +-sun.misc.Launcher$AppClassLoader@18b4aac2
```

1. 第二步，使用`AppClassLoader`加载`org.springframework.boot.SpringApplication`

```Shell
[arthas@52461]$ classloader -c 18b4aac2 --load org.springframework.boot.SpringApplication

load class success.

 class-info        org.springframework.boot.SpringApplication

 code-source       /Users/zhangyq01/Documents/.m2/org/springframework/boot/spring-boot/1.5.13.RELEASE/spring-boot-1.5.13.RELEASE.jar

 name              org.springframework.boot.SpringApplication

 isInterface       false

 isAnnotation      false

 isEnum            false

 isAnonymousClass  false

 isArray           false

 isLocalClass      false

 isMemberClass     false

 isPrimitive       false

 isSynthetic       false

 simple-name       SpringApplication

 modifier          public

 annotation

 interfaces

 super-class       +-java.lang.Object

 class-loader      +-sun.misc.Launcher$AppClassLoader@18b4aac2

                     +-sun.misc.Launcher$ExtClassLoader@710726a3

 classLoaderHash   18b4aac2
```

1. 第三步就可以`trace`程序入口，这里需要注意！一定要先运行第二步手动加载，否则就会得到下面的错误

```Shell
[arthas@57432]$ trace org.springframework.boot.SpringApplication run



Affect(class count: 0 , method count: 0) cost in 9 ms, listenerId: 1

No class or method is affected, try:

1. Execute sm CLASS_NAME METHOD_NAME to make sure the method you are tracing actually exists (it might be in your parent class).

2. Execute options unsafe true, if you want to enhance the classes under the java.* package.

3. Execute reset CLASS_NAME and try again, your method body might be too large.

4. Check arthas log: /Users/zhangyq01/logs/arthas/arthas.log

5. Visit https://github.com/alibaba/arthas/issues/47 for more details.
```

原因是因为程序还在`sleep`中，`Spring`的类还没被加载，所以需要手动加载，下面是`trace`结果

![image-20211220180030819](/images/oss/picGo/image-20211220180030819.png)

问题主要出在图中标记的两个方法中，分别占用了26秒和219秒，这两个加起来就260秒了。接下来开始分析，先`trace`第一个方法（重启应用并重复上述步骤）

```Shell
[arthas@61462]$ trace org.springframework.boot.SpringApplication prepareContext '#cost>20000'
```

![image-20211220180043210](/images/oss/picGo/image-20211220180043210.png)

继续`trace` * n，最后定位到一个小范围，然后就可以通过`review`代码或者`debug`的方式进行排查了，最终定位到该方法运行缓慢的问题：由于项目中缺少一个配置，所以默认访问了`localhost`，又一直得不到响应，所以会一直卡住。

第二个方法的排查过程也类似，这里不再具体说明。最终，优化后的项目启动速度为`22`秒，降低了`90%`

![image-20211220180203007](/images/oss/picGo/image-20211220180203007.png)

### 实践三：线上RPC接口调用错误分析

背景：外部系统调用此接口批量更新数据，接口中会判断是否可以处理，如果不能处理则交给`B`系统进行处理

此部分涉及到的表和代码（已简化）

数据库表

```Java
CREATE TABLE acceptance_record_batch_info (

  id bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',

  record_code varchar(32) NOT NULL DEFAULT '' COMMENT '验收记录号',

  batch_code varchar(64) NOT NULL DEFAULT '' COMMENT '批次编号',

  UNIQUE KEY uniq_record_code (record_code,batch_code) 

) ENGINE=InnoDB;
```

代码

```Java
// 新流程执行

reqList.forEach(req -> {

   //插入此条数据，如果无法处理会将其记录

   BatchUpdateAcceptanceOrderVO notRecord = updateAcceptanceOrder(req);

   if (null != notRecord) {

      qmsList.add(notRecord);

   }

});



// 请求Qms操作

if (!CollectionUtils.isEmpty(qmsList)) {

    //交给B系统处理

   qmsService.batchUpdateAcceptanceOrder(qmsList);

}
```

报错现象如下

```Java
### Error updating database. Cause: com.mysql.jdbc.exceptions.jdbc4.MySQLIntegrityConstraintViolationException: Duplicate entry 'YSJL00000002-PC-YXTEST-00118101' for key 'uniq_record_code'

### The error may involve defaultParameterMap

### The error occurred while setting parameters

### SQL: INSERT INTO acceptance_record_batch_info (record_code ,batch_code) values (?,?)

### Cause: com.mysql.jdbc.exceptions.jdbc4.MySQLIntegrityConstraintViolationException: Duplicate entry 'YSJL00000002-PC-YXTEST-00118101' for key 'uniq_record_code'
```

根据报错日志猜想可能是由于某种原因导致了重复数据产生，先删除掉该错误数据避免主流程阻塞。但是删除掉重复数据之后，接口还是一直报错，所以首先`trace`链路看看问题到底出在哪里

```Shell
[arthas]$ trace ServiceName methodName
```

`trace`结果中显示问题出在`qmsService.batchUpdateAcceptanceOrder(qmsList)`，也就是对于外部系统的调用中，如下

```Java
`---[103.652239ms] QmsService:batchUpdateAcceptanceOrder() [throws Exception]
```

这时问题就可以初步确定了：数据插入后，由于是批量操作，这一批中只要有一条数据错误，由于没有事务处理，会导致这批数据都变成脏数据。

下一步就是`watch`具体的方法

`QmsService.batchUpdateAcceptanceOrder()`方法实现如下

```Java
public void batchUpdateAcceptanceOrder(List reqList) {

    if(!CollectionUtils.isEmpty(reqList)){
        reqList.stream().forEach(item -> this.updateAcceptanceOrder(item));
    }
}

public int updateAcceptanceOrder(InputAcceptanceRequestDTO requestDto) {

    //查询是否有此条数据

    AcceptanceOrder acceptanceOrder = baseMapper.selectOne((new QueryWrapper<AcceptanceOrder>().lambda().eq(AcceptanceOrder::getSkuCode, requestDto.getSkuCode())

            .eq(AcceptanceOrder::getInputNo, requestDto.getInputNo()).eq(AcceptanceOrder::getDelFlag, DelFlagEnum.DEL_NO.getCode())

            .eq(AcceptanceOrder::getEnableFlag, EnableFlagEnum.NORMAL.getCode()).eq(AcceptanceOrder::getJudgeStatus, JudgeStatusEnum.OK.getCode())));

    if(null==acceptanceOrder){

        log.error("...");

        throw new ServiceException("...");

    }

    if (1!= baseMapper.updateAcceptanceOrder(requestDto)) {

        log.error("...");

        throw new ServiceException("...");

    }

}
```

根据方法参数 + 返回值 + 代码`review`方式定位问题，首先监测`mapper`层方法，这里有两个目的：如果执行到了此方法，可以利用入参排查更新失败的原因。如果没有执行到此方法，那么就是前面查询的问题

```Shell
[arthas]$ watch com.mryx.qms.dao.AcceptanceOrderDao updateAcceptanceOrder '{params[0], returnObj}'
```

监测后，发现并没有执行到此方法，所以可以初步认定是前面的查询结果为空导致的抛出异常

再次`watch`此方法的参数

```Java
[arthas]$ watch com.mryx.qms.service.impl.manual.NewAcceptanceOrderServiceImpl updateAcceptanceOrder '{params[0], returnObj}'
```

拿到请求参数后，根据请求参数拼接`SQL`去数据库中查询发现查询结果为空。通过`review`代码最终发现问题：此数据在数据库中是存在的，但是在其它逻辑中状态值已经被修改了，而查询时加入了`EnableFlag`状态值的查询条件导致此条数据被过滤掉，方法执行结果异常从而整条链路阻塞。



这部分主要通过几个实践以更全面的视角来介绍`Arthas`。总之，通过各种命令的组合，会实现很多不可思议的功能

`Arthas`虽然排查问题很方便，但是在日常的使用中，有两个注意事项

1. 由于`Arthas`是修改字节码实现，所以会很影响性能！线上使用一定要小心
2. 使用完成之后一定要用`reset`命令恢复被修改的字节码

## 总结

​	本文从基本介绍、核心原理、实际案例等各方面介绍了`Arthas`。可以看到，`Arthas`可以很大程度的提高线上问题的定位与处理效率。但是做为一款解决问题的工具，目前还缺少公司级别的落地方案，如权限控制、数据安全、操作简化等。将来会持续调研此部分，如果可能将在公司内推广使用。

