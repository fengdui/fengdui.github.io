---
title: "使用Velocity模板引擎渲染短信模板"
date: "2019-08-02"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


短信模板类
```
public class SmsTemplate {
    private String templateName;
    private String templateContent;
    private Map<String, Object> variables;
    
    // 构造函数、getter、setter
}
```
Velocity模板引擎配置
```
@Component
public class VelocityTemplateEngine {
    
    private VelocityEngine velocityEngine;
    
    @PostConstruct
    public void init() {
        VelocityEngine velocityEngine = new VelocityEngine();
        velocityEngine.setProperty(RuntimeConstants.RESOURCE_LOADER, "classpath");
        velocityEngine.setProperty("classpath.resource.loader.class", 
            ClasspathResourceLoader.class.getName());
        velocityEngine.init();
        this.velocityEngine = velocityEngine;
    }
    
    public String renderTemplate(String templateName, Map<String, Object> context) {
        Template template = velocityEngine.getTemplate("templates/" + templateName + ".vm");
        StringWriter writer = new StringWriter();
        template.merge(new VelocityContext(context), writer);
        return writer.toString();
    }
}
```
Velocity模板文件
```
【${companyName}】您的验证码是：${code}，有效期${expireMinutes}分钟，请勿泄露给他人。
```
短信服务实现
```
@Service
public class SmsService {
    
    @Autowired
    private VelocityTemplateEngine templateEngine;
    
    @Autowired
    private SmsClient smsClient; // 第三方短信服务客户端
    
    /**
     * 发送验证码短信
     */
    public boolean sendVerificationCode(String phone, String code, int expireMinutes) {
        Map<String, Object> context = new HashMap<>();
        context.put("companyName", "您的公司名");
        context.put("code", code);
        context.put("expireMinutes", expireMinutes);
        
        String content = templateEngine.renderTemplate("verification_code", context);
        return smsClient.sendSms(phone, content);
    }
    
    /**
     * 发送订单通知
     */
    public boolean sendOrderNotification(String phone, String orderNo, 
                                       String customerName, String status, 
                                       BigDecimal amount) {
        Map<String, Object> context = new HashMap<>();
        context.put("companyName", "您的公司名");
        context.put("customerName", customerName);
        context.put("orderNo", orderNo);
        context.put("status", status);
        context.put("amount", amount);
        
        String content = templateEngine.renderTemplate("order_notification", context);
        return smsClient.sendSms(phone, content);
    }
    
    /**
     * 通用短信发送方法
     */
    public boolean sendSms(String phone, String templateName, Map<String, Object> params) {
        String content = templateEngine.renderTemplate(templateName, params);
        return smsClient.sendSms(phone, content);
    }
}
```
模板管理服务
```
@Service
public class TemplateManagementService {
    
    @Autowired
    private VelocityTemplateEngine templateEngine;
    
    /**
     * 预览模板渲染结果
     */
    public String previewTemplate(String templateName, Map<String, Object> testData) {
        return templateEngine.renderTemplate(templateName, testData);
    }
    
    /**
     * 验证模板语法
     */
    public boolean validateTemplate(String templateContent) {
        try {
            VelocityEngine tempEngine = new VelocityEngine();
            tempEngine.init();
            
            StringReader reader = new StringReader(templateContent);
            Template template = new Template();
            template.setData(reader);
            template.initDocument();
            
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}
```