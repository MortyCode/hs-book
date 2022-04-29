# JAVA探针技术（二）

## 探针修改Class的限制

#### 主程序运行前的Agent

* 出名称以外，可以更改任意内容，名称改了，ClassLoad 就会出问题

#### 主程序运行中的Agent

* 不能修改Class的文件结构，即不能添加方法，不能添加字段，只能修改方法体的内容，否则就会报
  * UnsupportedOperationException: **class redefinition failed**: **attempted to change the schema (add/remove fields) 类似的异常**

## 两种修改类的方式

### 修改类的方式

#### redefineClasses：

* 重新定义class
* 自身提供的Class字节码替换掉已存在的Class
* 应用于线上debug的时候比较方便

#### retransformClasses：

* 修改class
* 在已存在的Class字节码上修改后再进行替换，类似于对Class进行包装。
* 应用于通用aop服务的时候比较方便

## redefineClasses

* 可以直接采用指定文件进行读取，然后直接进行替换
* 一般实现的方式是下面这种方式

```java
byte[] classBytes = Files.readAllBytes(Paths.get(classFile));
ClassDefinition classDefinition = new ClassDefinition(redefineClass, classBytes);
inst.redefineClasses(classDefinition);
```

## retransformClasses

* retransformClasses的使用需要Transformer类的配合，使用Transformer的包装对Class进行包装，然后替换

### ClassFileTransformer

* 对类进行包装的转换类

#### 类定义

```java
public interface ClassFileTransformer {
    byte[] transform(  ClassLoader         loader,
                String              className,
                Class<?>            classBeingRedefined,
                ProtectionDomain    protectionDomain,
                byte[]              classfileBuffer)
        throws IllegalClassFormatException;
}
```

```json
loader –  
要转换的类的定义加载程序，如果引导加载程序

className–  
Java虚拟机规范中定义的完全限定类和接口名称的内部形式的类名称。例如，“java/util/List”。

classBeingRedefined–
如果这是由重定义或重传触发的，则被重定义或重传的类；如果这是类加载，则为null

protectionDomain–
正在定义或重新定义的类的保护域

classfileBuffer——
类格式的输入字节缓冲区——不得修改
```

#### 实现方式

* 一般的实现方式是在**transform**方法中，一般是使用ASM，javassist之类的字节码操纵技术对字节码进行包装。

```java
    @Override
    public byte[] transform(ClassLoader loader,
                            //要转换的类的定义加载程序，
                            String className,
                            //Java虚拟机规范中定义的完全限定类和接口名称的内部形式的类名称。
                            // 例如，“java/util/List”。
                            Class<?> classBeingRedefined,
                            //如果这是由重定义或重传触发的，则被重定义或重传的类；如果这是类加载，则为null
                            ProtectionDomain protectionDomain,
                            //正在定义或重新定义的类的保护域
                            byte[] classfileBuffer
                            //类格式的输入字节缓冲区——不得修改
    ) throws IllegalClassFormatException {
        ClassReader classReader = new ClassReader(classfileBuffer);
        PreClassVisitor preClassVisitor = new PreClassVisitor( new ClassWriter(ClassWriter.COMPUTE_MAXS));
        classReader.accept(preClassVisitor,0);
        return preClassVisitor.toByteArray();
    }
```
