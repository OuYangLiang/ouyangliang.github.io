---
layout: post
title:  "基于Java Agent与Javassist实现零侵入AOP"
date:   2019-03-07 08:00:00 +0800
categories: java
keywords: aop,agent,instrument,javassist
description: 基于Java Agent与Javassist实现零侵入AOP
commentId: 2019-03-07
---
Java SE 5引入了一个静态Instrument的概念，利用它我们可以构建一个独立于应用程序的代理程序（Agent），用来监测和协助运行在JVM上的程序，可以在程序启动前修改类的定义。有了这样的功能，我们就可以实现更为灵活的运行时虚拟机监控和Java类操作了，这样的特性实际上提供了一种虚拟机级别支持的AOP实现方式，使得开发者无需对应用程序做任何升级和改动，就可以实现某些AOP的功能了。

在应用启动时，通过<kbd>-javaagent</kbd>参数指定一个代理程序，实现对类的一些修改。大概的步骤如下：

**一、编写一个`premain`函数**

```java
public static void premain(String agentArgs, Instrumentation inst);
public static void premain(String agentArgs);
```

在`premain`函数中，可以进行对类的各种操作。`agentArgs`是`premain`函数得到的程序参数，随同 “–javaagent”一起传入。`inst`是一个`java.lang.instrument.Instrumentation`的实例，由JVM自动传入，是instrument包中定义的一个接口，也是这个包的核心部分，集中了其中几乎所有的功能方法，例如类定义的转换和操作等等。

**二、jar打包**

将包含`premain`函数的类打包成一个jar文件，并在其中的manifest属性当中加入Premain-Class来指定带有`premain`的Java类。

**三、运行程序**

```shell
java -javaagent:jar[=agentArgs] -cp 应用程序jar 主类
```

<br/>

### 一个示例

现在假设我们有一个应用程序，你在浏览器访问一个地址，页面会返回你一个提示信息。比如你访问：http://localhost:8080/hello/ouyang，页面返回Hello ouyang，如下所示：

<center><img src="{{site.baseurl}}/pic/instrument/1.png" width="60%"/></center>

程序使用springboot搭建，主要代码如下所示：

<img src="{{site.baseurl}}/pic/instrument/0.png" width="100%"/>

<br/>

现在我们想要对程序做一些修改，比如统计`format`方法的耗时，看看如何利用Instrument实现零侵入来实现这个功能。首先，我们另启一个java工程，并编写一个premain方法的类：

```java
package com.personal.oyl.instrument;

import java.lang.instrument.Instrumentation;

public class TheAgent {
    public static void premain(String agentArgs, Instrumentation inst) {
        inst.addTransformer(new TheClassFileTransformer());
    }
}
```

`addTransformer`方法接收一个`ClassFileTransformer`对象，它实现了`transform`方法，主要目的就是对类的字节码进行修改。程序启动时，`premain`函数会在`main`之前执行，这里相当于注册了一个类的转换器吧。之后在`main`函数执行前，JVM每装载一个类，`ClassFileTransformer`对象的`transform`方法就会执行一次，对类进行转换。

现在的问题就是需要实现一个`ClassFileTransformer`，在`transform`方法中通过对类的字节码进行扩展和修改，来实现我们的需求。这里我们使用了javassist，一个开源的操作Java字节码的类库。本文不会介绍关于javassist的用法，如果你感兴趣，可以看看[这里](http://www.javassist.org/tutorial/tutorial.html)。

> 类似字节码操作方法还有ASM。在性能上Javassist高于反射，但低于ASM，因为Javassist增加了一层抽象。在实现成本上Javassist和反射都很低，而ASM由于直接操作字节码，相比Javassist源码级别的API实现成本高很多。

```java
public class TheClassFileTransformer implements ClassFileTransformer {
    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined,
            ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        if (!className.startsWith("com/ymm/crm/sample/demo")) { // 只转换我们程序的代码，其它类不做处理
            return null;
        }
        String newClassName = className.replace("/", ".");
        System.out.println("Transforming: " + newClassName);

        ClassPool pool = ClassPool.getDefault();
        CtClass cl = null;
        try {
            pool.insertClassPath(new LoaderClassPath(loader));
            try {
                cl = pool.get(newClassName);
            } catch (NotFoundException e) {
                // Spring会能过字节码对我们的类进行扩展，这种情况下找不到对应的class文件，会报NotFoundException
                // 这时我们可以直接读取classfileBuffer，直接通过类的字节码创建CtClass对象
                ByteArrayInputStream is = null;
                try {
                    is = new ByteArrayInputStream(classfileBuffer);
                    cl = pool.makeClass(is);
                } finally {
                    if (null != is) {
                        is.close();
                        is = null;
                    }
                }
            }

            if (cl.isInterface()) {
                return null;
            }

            CtMethod[] methods = cl.getDeclaredMethods();
            for (CtMethod method : methods) {
                enhance(method);
            }
            return cl.toBytecode();
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        } finally {
            if (null != cl) {
                cl.detach();
            }
        }
    }

    private void enhance(CtMethod method) throws CannotCompileException {
        method.insertBefore("{ System.out.println(\"" + method.getLongName() + " called ...\"); }");
        method.instrument(new ExprEditor() {
            public void edit(MethodCall m) throws CannotCompileException {
                if (m.getClassName().startsWith("com.ymm.crm.sample.demo.service")) {
                    StringBuilder sb = new StringBuilder();
                    sb.append("{long startTime = System.currentTimeMillis();");
                    sb.append("$_=$proceed($$) + \" amended...\";");
                    sb.append("System.out.println(\"");
                    sb.append(m.getClassName()).append(".").append(m.getMethodName());
                    sb.append(" cost: \" + (System.currentTimeMillis() - startTime) + \" ms\");}");
                    m.replace(sb.toString());
                }
            }
        });
    }
}
```

前面我们说了，JVM每装载一个类，`transform`方法就会被回调。它先检查要加载的类是否是我们应用程序的类（通过`className.startsWith("com/ymm/crm/sample/demo")`判断，如果不是，就不做任何处理。如果是我们自己的类，就通过类名读取对应的class文件，生成一个`CtClass`对象，并遍历和加强`CtClass`对应的类的每个方法。

`TheClassFileTransformer`的重点是`enhance`方法，它在对应方法的前面插入了一个语句块，这样每个方法被调用时，都会打印一行信息，输出方法名加上" called ..."。另外，如果被处理的方法所属的类是在`com.ymm.crm.sample.demo.service`包下，还修改了方法的返回值，在原返回值的基础上加上了" amended..."，同时会在控制台上输出方法的执行耗时。

由于我们的Agent程序依赖了javassist，打包的时候可以利用maven的assembly插件将所有内容打在一起。现在我们加上agent运行程序看一下，应用程序是通过springboot搭建的，所以可以使用下面的命令格式启动：

```shell
java -javaagent:agent.jar -jar app.jar
```

现在访问：http://localhost:8080/hello/ouyang，页面会返回Hello ouyang amended...，如下所示：

<center><img src="{{site.baseurl}}/pic/instrument/2.png" width="60%"/></center>

从控制台上可以看到，DemoApplication.hello与ServiceImpl.format方法都打印了方法名+" called ..."，另外还输出了service包下的ServiceImpl.format方法的耗时。

<center><img src="{{site.baseurl}}/pic/instrument/3.png" width="90%"/></center>

第一次了解到java agent时，真的被惊呀了，居然还能这么玩。对于运维、监控来说，真的是太强大了，可以真正无入侵的实现打点、性能监控、流量复制等，只有你想不到，没有它做不到的啊。
