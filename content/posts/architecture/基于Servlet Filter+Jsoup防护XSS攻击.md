---
title: "基于Servlet Filter+Jsoup防护XSS攻击"
date: "2015-10-08"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

XSS过滤器 (XssFilter)
```
@Override
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
    HttpServletRequest httpRequest = (HttpServletRequest) request;
    
    // 检查是否需要过滤
    if (shouldNotFilter(httpRequest)) {
        chain.doFilter(request, response);
        return;
    }
    
    // 使用XSS包装器过滤请求
    chain.doFilter(new XssHttpServletRequestWrapper(httpRequest), response);
}
```
请求包装器 (XssHttpServletRequestWrapper)
```
public class XssHttpServletRequestWrapper extends HttpServletRequestWrapper {

    public XssHttpServletRequestWrapper(HttpServletRequest request) {
        super(request);
    }

    @Override
    public ServletInputStream getInputStream() throws IOException {
        // 非json类型，直接返回
        if (!MediaType.APPLICATION_JSON_VALUE.equalsIgnoreCase(super.getHeader(HttpHeaders.CONTENT_TYPE))) {
            return super.getInputStream();
        }

        // 为空，直接返回
        String json = IoUtil.readUtf8(super.getInputStream());
        if (StringUtils.isBlank(json)) {
            return super.getInputStream();
        }

        // xss过滤
        json = xssEncode(json);
        final ByteArrayInputStream bis = new ByteArrayInputStream(json.getBytes(StandardCharsets.UTF_8));
        return new ServletInputStream() {
            @Override
            public boolean isFinished() {
                return true;
            }

            @Override
            public boolean isReady() {
                return true;
            }

            @Override
            public void setReadListener(ReadListener readListener) {

            }

            @Override
            public int read() {
                return bis.read();
            }
        };
    }

    @Override
    public String getParameter(String name) {
        String value = super.getParameter(xssEncode(name));
        if (StringUtils.isNotBlank(value)) {
            value = xssEncode(value);
        }
        return value;
    }

    @Override
    public String[] getParameterValues(String name) {
        String[] parameters = super.getParameterValues(name);
        if (parameters == null || parameters.length == 0) {
            return null;
        }

        for (int i = 0; i < parameters.length; i++) {
            parameters[i] = xssEncode(parameters[i]);
        }
        return parameters;
    }

    @Override
    public Map<String, String[]> getParameterMap() {
        Map<String, String[]> map = new LinkedHashMap<>();
        Map<String, String[]> parameters = super.getParameterMap();
        for (String key : parameters.keySet()) {
            String[] values = parameters.get(key);
            for (int i = 0; i < values.length; i++) {
                values[i] = xssEncode(values[i]);
            }
            map.put(key, values);
        }
        return map;
    }

    @Override
    public String getHeader(String name) {
        String value = super.getHeader(xssEncode(name));
        if (StringUtils.isNotBlank(value)) {
            value = xssEncode(value);
        }
        return value;
    }

    private String xssEncode(String input) {
        return XssUtils.filter(input);
    }

}
```
拦截并过滤以下内容：  
JSON请求体: getInputStream()  
URL参数: getParameter(), getParameterValues(), getParameterMap()  
请求头: getHeader()  
使用Jsoup库进行HTML清理  

```
private String cleanHtml(String input) {
    return Jsoup.clean(input, xssWhitelist());
}
/**
 * XSS过滤白名单
 */
private static Safelist xssWhitelist() {
    return new Safelist()
            // 支持的标签
            .addTags("a", "b", "blockquote", "br", "caption", "cite", "code", "col", "colgroup", "dd", "div", "dl",
                    "dt", "em", "h1", "h2", "h3", "h4", "h5", "h6", "i", "img", "li", "ol", "p", "pre", "q",
                    "small",
                    "strike", "strong", "sub", "sup", "table", "tbody", "td", "tfoot", "th", "thead", "tr", "u",
                    "ul",
                    "embed", "object", "param", "span")

            // 支持的标签属性
            .addAttributes("a", "href", "class", "style", "target", "rel", "nofollow")
            .addAttributes("blockquote", "cite")
            .addAttributes("code", "class", "style")
            .addAttributes("col", "span", "width")
            .addAttributes("colgroup", "span", "width")
            .addAttributes("img", "align", "alt", "height", "src", "title", "width", "class", "style")
            .addAttributes("ol", "start", "type")
            .addAttributes("q", "cite")
            .addAttributes("table", "summary", "width", "class", "style")
            .addAttributes("tr", "abbr", "axis", "colspan", "rowspan", "width", "style")
            .addAttributes("td", "abbr", "axis", "colspan", "rowspan", "width", "style")
            .addAttributes("th", "abbr", "axis", "colspan", "rowspan", "scope", "width", "style")
            .addAttributes("ul", "type", "style")
            .addAttributes("pre", "class", "style")
            .addAttributes("div", "class", "id", "style")
            .addAttributes("embed", "src", "wmode", "flashvars", "pluginspage", "allowFullScreen",
                    "allowfullscreen",
                    "quality", "width", "height", "align", "allowScriptAccess", "allowscriptaccess",
                    "allownetworking", "type")
            .addAttributes("object", "type", "id", "name", "data", "width", "height", "style", "classid",
                    "codebase")
            .addAttributes("param", "name", "value")
            .addAttributes("span", "class", "style")

            // 标签属性对应的协议
            .addProtocols("a", "href", "ftp", "http", "https", "mailto")
            .addProtocols("img", "src", "http", "https")
            .addProtocols("blockquote", "cite", "http", "https")
            .addProtocols("cite", "cite", "http", "https")
            .addProtocols("q", "cite", "http", "https")
            .addProtocols("embed", "src", "http", "https");
}
````
脚本注入防护: 过滤 <script> 等危险标签  
HTML注入防护: 只允许白名单内的安全标签和属性  
协议限制: 只允许 http/https 等安全协议  
全面覆盖: 拦截所有HTTP请求的参数、请求体、请求头  