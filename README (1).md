# JAVA探针技术(一)

JavaAgent是一个JVM插件，它能够利用jvm提供的 Instrumentation API（Java1.5开始提供）实现字节码修改的功能。Agent分为2种：主程序运行前的Agent，主程序之后运行的Agent（Jdk1.6增加）。 JavaAgent常用于 代码热更新，AOP，JVM监控等功能。

## 主程序运行前的Agent

### 1. 编写探针程序

* 名称必须为premain
* 参数可以是premain(String agentOps, Instrumentation inst) 也可以是premain(String agentOps)
* 优先执行premain(String agentOps)

```json
public class AgentTest {

    该方法在main方法之前运行，与main方法运行在同一个JVM中
    并被同一个System ClassLoader装载
    被统一的安全策略(security policy)和上下文(context)管理
    public static void  premain(String agentOps, Instrumentation inst){
        System.out.println("====premain 方法执行");
        System.out.println("参数为:"+agentOps);
    }

    如果不存在 premain(String agentOps, Instrumentation inst)
    则会执行 premain(String agentOps)
    public static void premain(String agentOps){

        System.out.println("====premain方法执行2====");
        System.out.println(agentOps);
    }
}
```

### 2. 在MANIFEST.MF 配置环境参数

* 普通项目这样配置

```json
Manifest-Version: 1.0
Premain-Class: com.agent.AgentTest 
Can-Redefine-Classes: true 
Can-Retransform-Classes: true 
```

* 可以配置的属性

```json
Premain-Class	指定代理类
Agent-Class	指定代理类
Boot-Class-Path	指定bootstrap类加载器的搜索路径，在平台指定的查找路径失败的时候生效， 可选
Can-Redefine-Classes	是否需要重新定义所有类，默认为false，可选。
Can-Retransform-Classes	是否需要retransform，默认为false,可选
```

* maven项目这样配置
* manifestEntries里面的元素就与上面的配置对应

```json
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>2.4</version>
                <configuration>
                    <archive>
                        <manifest>
                            <addClasspath>true</addClasspath>
                        </manifest>

                        <manifestEntries>
                            <Premain-Class>
                                com.agent.AgentTest
                            </Premain-Class>

                            <Can-Redefine-Classes>
                                true
                            </Can-Redefine-Classes>

                            <Can-Retransform-Classes>
                                true
                            </Can-Retransform-Classes>

                            <Manifest-Version>
                                true
                            </Manifest-Version>

                        </manifestEntries>
                    </archive>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

### 3. 使用探针程序

* 打包探针项目 为 preAgent.jar
* 启动主函数的时候添加jvm参数
* params就对应premain函数中 agentOps参数

```json
-javaagent: 路径/preAgent.jar=params
```

## 主程序之后运行的Agent

启动前探针使用方式比较局限，而且每次探针更改的时候，都需要重新启动应用，而主程序之后的探针程序就可以直接连接到已经启动的jvm中。 可以实现例如动态替换类，查看加载类信息的一些功能。

### 实现一个指定动态类替换的功能

* 下面就实现一个指定类，指定class文件动态替换，实现动态日志增加的功能。

### 1. 编写探针程序

* 主程序后的探针程序名称必须为agentmain
* 通过agentOps参数将需要替换的类名和Class类文件路径传递进来
* 然后获取全部加载的Class去，通过类名筛选出来要替换的Class
* 通过传递进行的Class类文件路径加载数据
* 通过redefineClasses进行类文件的热替换
* 使用redefineClasses函数必须将Can-Redefine-Classes环节变量设置为true

```json
public static void  agentmain(String agentOps, Instrumentation inst){
        System.out.println("====agentmain 方法开始");

        String[] split = agentOps.split(",");

        String className = split[0];
        String classFile = split[1];

        System.out.println("替换类为:   "+className);

        Class<?> redefineClass = null;
        Class<?>[] allLoadedClasses = inst.getAllLoadedClasses();
        for (Class<?> clazz : allLoadedClasses) {
            if (className.equals(clazz.getCanonicalName())){
                redefineClass = clazz;
            }
        }

        if (redefineClass==null){
            return;
        }

        //热替换
        try {
            byte[] classBytes = Files.readAllBytes(Paths.get(classFile));
            ClassDefinition classDefinition = new ClassDefinition(redefineClass, classBytes);
            inst.redefineClasses(classDefinition);
        } catch (ClassNotFoundException | UnmodifiableClassException | IOException e) {
            e.printStackTrace();
        }
        System.out.println("====agentmain 方法结束");
    }
```

### 2. 在MANIFEST.MF 配置环境参数

* 普通项目这样配置

```json
Manifest-Version: 1.0
Agent-Class: com.agent.AgentDynamic
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

* maven项目这样配置
* manifestEntries里面的元素就与上面的配置对应

```json
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>2.4</version>
                <configuration>
                    <archive>
                        <manifest>
                            <addClasspath>true</addClasspath>
                        </manifest>

                        <manifestEntries>
                            <Agent-Class>
                                com.agent.AgentDynamic
                            </Agent-Class>

                            <Can-Redefine-Classes>
                                true
                            </Can-Redefine-Classes>

                            <Can-Retransform-Classes>
                                true
                            </Can-Retransform-Classes>

                            <Manifest-Version>
                                true
                            </Manifest-Version>

                        </manifestEntries>
                    </archive>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

### 3. 使用探针程序

* 先使用 **jps** 指令 或者 **ps -aux|grep java** 找到目标JVM线程ID
* 编写使用探针程序
  * 将目标线程attach到VirtualMachine
  * 配置参数agentOps ,加载探针，此时就会执行探针中的程序
  * 通过VirtualMachine还能获取到对应JVM的系统参数，以及探针的一些参数

```json
    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {
        VirtualMachine target = VirtualMachine.attach("96003");//目标VM线程ID

        String agentOps = "com.api.rcode.controller.HomeController,/Users/hailingliang/app/workspace/jave-code-note/server-api/target/classes/com/api/rcode/controller/HomeController.class";
        target.loadAgent("/Users/hailingliang/app/workspace/jave-code-note/java-learn/agent/target/agent-1.0-SNAPSHOT.jar",
                agentOps);

        Properties agentProperties = target.getAgentProperties();
        System.out.println(agentProperties);

        Properties systemProperties = target.getSystemProperties();
        System.out.println(systemProperties);

        target.detach();
    }
```

### 4. 结果展示

* 通过这个方法，我们就可以实现在运行时，对Class文件的动态修改替换

```json
@RestController("/home")
public class HomeController {
    @RequestMapping("/h2")
    public String h2(boolean p1, int p2){
        System.out.println(p1+" "+p2);
        System.out.println("这个是热加载的效果");
        return "h2 p1: "+p1+" p2:"+p2;
    }
}
```

```json
====agentmain 方法开始
参数为:   com.api.rcode.controller.HomeController,/Users/hailingliang/app/workspace/jave-code-note/server-api/target/classes/com/api/rcode/controller/HomeController.class
objectSize   1040
====agentmain 方法完成
true 122112
这个是额外加的一段话12312312312132123
====agentmain 方法开始
参数为:   com.api.rcode.controller.HomeController,/Users/hailingliang/app/workspace/jave-code-note/server-api/target/classes/com/api/rcode/controller/HomeController.class
objectSize   1040
====agentmain 方法完成
true 122112
这个是热加载的效果
```

##
