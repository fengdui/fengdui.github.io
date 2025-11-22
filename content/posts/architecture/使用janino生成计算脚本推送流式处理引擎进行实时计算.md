---
title: "使用janino生成计算脚本推送流式处理引擎进行实时计算"
date: "2017-10-20"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

用户在页面上可视化的配置某个主体在某个时间范围内怎么计算 最后生成一个k/v给规则平台  
比如用户配置 某个商家 在过去24小时内的 总交易金额  
这里就会使用janino生成一段java代码 代码里面包括结果的key是什么 计算的时间范围 结果是每笔订单求和  
当交易过来的时候 流式处理引擎根据这段java代码去计算  

用户配置完成之后生成java代码片段  
```
return CalcTaskBuilder.getInstance("CUST_ID")
    .selectMemCachedItems("PAYMENT", "CREDITCARD", TimedItems.parse("86400"))
    .lazyGroup(true)
    .selectObjectType(PaymentTransaction.class)
    .objectKey(new Key() {
      public String key(Object obj) {
        return String.valueOf(((PaymentTransaction) obj).getTransAmt());
      }
    })
    .objectCond(new Cond() {
      public boolean cond(Object obj) {
        PaymentTransaction pojo = (PaymentTransaction) obj;
        return eq(pojo.getStatus(), "SUCCESS");
      }
    })
    .sTermNotNull(PaymentTransaction, "transAmt")
    .expirePattern("7d")
    .varDate(new Variable < Date > () {
      public Date select(Object obj) {
        return getTimeFunction(obj);
      }
    })
    .method(new Method() {
      public Mergeable invoke(Object obj) {
        PaymentTransaction pojo = (PaymentTransaction) obj;
        return new SumNumber((BigDecimal)pojo.getTransAmt());
      }
    })
    .build();
```
通过执行这个java代码 最后得到一个Mergeable 里面包含一个treeMap 每个key是时间戳 每个value是计算结果  
如果不是第一次计算 会根据objectKey去缓存里面查询之前的计算结果 这个map就不是为空  
如果是7d 就是7天 每天的数据做一个合并  
如果写成168h 也是7天 但是是每个小时的数据做一个合并 有168个kv  
当数据过来进行计算的时候 会做一个时间窗口滑动 只维护7天的数据 最早的key淘汰 key上有时间戳  

使用ClassBodyEvaluator输入拼接的字符串生成java代码  
使用URLClassLoader指定外部 JAR 包路径 需要依赖一些二方包三方包  
```
import org.codehaus.janino.ClassBodyEvaluator;
import java.io.File;
import java.net.URL;
import java.net.URLClassLoader;

public class JarLoaderExample {

    public static void main(String[] args) throws Exception {
        // 1. 指定外部 JAR 包路径
        File jarFile = new File("path/to/your/dependency.jar");
        
        // 2. 创建 URLClassLoader，将 JAR 包添加进去
        URLClassLoader classLoader = new URLClassLoader(
            new URL[]{jarFile.toURI().toURL()},
            Thread.currentThread().getContextClassLoader() // 使用当前类加载器作为父加载器
        );
        
        // 3. 创建 ClassBodyEvaluator 实例，并设置类加载器
        ClassBodyEvaluator evaluator = new ClassBodyEvaluator();
        evaluator.setParentClassLoader(classLoader); // 关键步骤！
        
        // 4. 编写使用了外部 JAR 中类的代码片段
        String code = 
            "import com.your.dependency.ExternalClass;\n" + // 确保导入
            "public class MyClass {\n" +
            "    public String process() {\n" +
            "        ExternalClass external = new ExternalClass();\n" +
            "        return external.doSomething();\n" +
            "    }\n" +
            "}";
        
        // 5. 编译并加载生成的类
        evaluator.cook(code);
        Class<?> generatedClass = evaluator.getClazz();
        Object instance = generatedClass.newInstance();
        
        // 调用方法
        String result = (String) instance.getClass().getMethod("process").invoke(instance);
        System.out.println("Result: " + result);
        
        // 6. 使用后，可根据情况考虑关闭 ClassLoader（如果后续不再需要）
        if (classLoader instanceof Closeable) {
            ((Closeable) classLoader).close();
        }
    }
}
```