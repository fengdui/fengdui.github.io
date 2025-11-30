---
title: "使用httpclient抓取题目进行编译"
date: "2016-05-13"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

ForwardingJavaFileManager 是 Java 编译器 API（javax.tools 包）中的一个类，它实现了 JavaFileManager 接口，  
并采用装饰器模式（Decorator Pattern）来包装另一个 JavaFileManager 实例  
用于实现内存中的 Java 代码编译功能  
```
import javax.tools.*;
import java.io.IOException;

public class MyFileManager extends ForwardingJavaFileManager {

    private ClassJavaFileObject fileObject;

    public ClassJavaFileObject getFileObject() {
        return fileObject;
    }

    public void setFileObject(ClassJavaFileObject fileObject) {
        this.fileObject = fileObject;
    }

    public MyFileManager(StandardJavaFileManager javaFileManager) {
        super(javaFileManager);
    }

    @Override
    public JavaFileObject getJavaFileForOutput(Location location,
                                               String className, JavaFileObject.Kind kind, FileObject sibling)
            throws IOException {
        fileObject = new ClassJavaFileObject(className, kind);
        return fileObject;
    }
}
```
编译之后放到map中
```
public String compileJava(String source) throws Exception {

    JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
    DiagnosticCollector<JavaFileObject> diagnostics = new DiagnosticCollector<JavaFileObject>();
    MyFileManager fileManager = new MyFileManager(compiler.getStandardFileManager(diagnostics, null, null));
    List<JavaFileObject> jfiles = new ArrayList<JavaFileObject>();
    jfiles.add(new SourceJavaFileObject("Main", source));

    List<String> options = new ArrayList<String>();
    options.add("-encoding");
    options.add("UTF-8");
    JavaCompiler.CompilationTask task = compiler.getTask(null, fileManager, diagnostics, options, null, jfiles);
    boolean success = task.call();

    if (success) {
        ClassJavaFileObject jco = fileManager.getFileObject();
        DynamicClassLoader dynamicClassLoader = new DynamicClassLoader();
        Class clazz = dynamicClassLoader.loadClass("Main", jco);
        ContextUtil.map.put("Main", clazz);
        return "编译成功";
    } else {
        String error = "";
        for (Diagnostic diagnostic : diagnostics.getDiagnostics()) {
            error = error + compilePrint(diagnostic);
        }
        return error;
    }
}
```
编译的code是通过httpclient作为爬虫抓取到的
```
public abstract class Crawler {

    public Problem crawl(String problemId) {
        try {

            String problemUrl = getProblemUrl(problemId);
            CloseableHttpClient httpClient = HttpClients.createDefault();
            HttpGet httpGet = new HttpGet(problemUrl);
            CloseableHttpResponse response = httpClient.execute(httpGet);
            Problem info = new Problem();
            populateProblemInfo(info, problemId, EntityUtils.toString(response.getEntity()));
            return info;
        }
        catch (Exception e) {
            return null;
        }
        finally {

        }
    }

    protected abstract String getProblemUrl(String problemId);

    protected abstract void populateProblemInfo(Problem info, String problemId, String html) throws Exception;

}
````
抓取之后正则处理
```
protected void populateProblemInfo(Problem info, String problemId, String html) throws Exception {
//        System.out.println("进入正则");
    info.title = RegUtil.regFind(html, "<h2><strong>([\\s\\S]*?)</strong>").trim();
    info.timeLimit = (Integer.parseInt(RegUtil.regFind(html, "/(\\d*)MS")));
    info.memoryLimit = (Integer.parseInt(RegUtil.regFind(html, "(\\d*)KByte")));
    info.description = (RegUtil.regFind(html, "Times New Roman\">([\\s\\S]*?)</font></p></div>"));
    info.input = (RegUtil.regFind2(html, "Times New Roman\">([\\s\\S]*?)</font></p></div>"));
    info.output = (RegUtil.regFind3(html, "Times New Roman\">([\\s\\S]*?)</font></p></div>"));
    info.sampleInput = (RegUtil.regFind(html, "sample_input\" class=\"sample_input_output\" readonly>([\\s\\S]*?)</textarea></p>"));
    info.sampleOutput = (RegUtil.regFind(html, "sample_output\" class=\"sample_input_output\" readonly>([\\s\\S]*?)</textarea></p>"));
//        System.out.println(info.title);
//        System.out.println(info.timeLimit);
//        System.out.println(info.memoryLimit);
//        System.out.println(info.description);
//        System.out.println(info.input);
//        System.out.println(info.output);
//        System.out.println(info.sampleInput);
//        System.out.println(info.sampleOutput);
}
```