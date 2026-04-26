---
title: "pageOffice+thymeleaf在线office处理"
date: "2026-04-26"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

PageOffice 与 Thymeleaf 的具体对接，核心思路是：在后端 Controller 中，通过 PageOffice 的 getHtmlCode 方法生成控件所需的 HTML 代码，  
然后将这段代码传给 Thymeleaf 模板页面，并在模板中使用 th:utext 语法将其“嵌入”到 <div> 容器中   

```
import com.zhuozhengsoft.pageoffice.OpenModeType;
import com.zhuozhengsoft.pageoffice.PageOfficeCtrl;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Controller
public class WordController {

    // 当用户访问 /openWord 时，打开在线编辑页面
    @GetMapping("/openWord")
    public String openWord(HttpServletRequest request, Model model) {
        // 1. 创建 PageOffice 控制器实例
        PageOfficeCtrl poCtrl = new PageOfficeCtrl(request);
        // 2. 设置服务页面，这个路径必须与启动类中配置的映射一致
        poCtrl.setServerPage(request.getContextPath() + "/poserver.zz");
        // 3. 设置保存文件的请求路径，对应下面的 saveFile 方法
        poCtrl.setSaveFilePage("/saveFile");
        // 4. 打开 Word 文件，此处需要替换为你服务器上真实的文件路径
        poCtrl.webOpen("D:\\documents\\test.docx", OpenModeType.docNormalEdit, "用户名");
        
        // 5. 获取 PageOffice 控件生成的 HTML 代码，并将其存入 Model，供 Thymeleaf 使用
        String poHtml = poCtrl.getHtmlCode("PageOfficeCtrl1");
        model.addAttribute("pageoffice", poHtml);
        
        // 6. 返回 Thymeleaf 模板的名称，即 src/main/resources/templates/word.html
        return "word";
    }

    // 这个方法专门用来接收 PageOffice 控件保存的文件
    @PostMapping("/saveFile")
    public void saveFile(HttpServletRequest request, HttpServletResponse response) {
        com.zhuozhengsoft.pageoffice.FileSaver fs = new com.zhuozhengsoft.pageoffice.FileSaver(request, response);
        try {
            // 将接收到的文件流保存到服务器，路径和文件名可根据业务逻辑处理
            fs.saveToFile("D:\\documents\\saved_" + fs.getFileName());
            // 保存成功后，必须调用此方法以通知 PageOffice 控件
            fs.close();
        } catch (Exception e) {
            // 如果保存失败，可通过 fs.setCustomSaveResult() 方法向控件返回错误信息
            fs.setCustomSaveResult("保存失败: " + e.getMessage());
            fs.close();
        }
    }
}
```

前端模版
```
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>PageOffice 在线编辑</title>
    <!-- 必须引用 pageoffice.js，路径来自于启动类中的映射配置 -->
    <script type="text/javascript" src="/pageoffice.js"></script>
</head>
<body>
    <div style="width: auto; height: 700px;" th:utext="${pageoffice}">
        <!-- PageOffice 控件将动态渲染在此 div 内 -->
    </div>
</body>
</html>
```