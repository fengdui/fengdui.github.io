---
title: "一种基于不可见字符的api水印的实现方式"
date: "2025-02-11"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


使用零宽字符实现隐形水印：
使用3个零宽字符：\u200B、\u200C、\u200D
16位长度的水印字符串
支持最大43046720个水印ID

比如水印id是x 将x转成3进制 为什么是三进制 因为我们只有3个不可见字符
然后每一位是是0就取第一个不可见字符 如果是1就取第二个不可见字符 如果是2就取第三个不可见字符 拼接起来加到原来的字符串后面


```
// 水印ID生成（1-65535范围内随机）
Random random = new Random();
int waterId = random.nextInt(65535);

private static final int WATERMARK_LENGTH = 16;

// 将水印ID转换为零宽字符串
private static String generateRandomWatermark(long watermarkId) {
    StringBuilder watermark = new StringBuilder();
    for (int i = 0; i < WATERMARK_LENGTH; i++) {
        int charIndex = (int) (watermarkId % WATERMARK_CHARS.length());
        watermarkId /= WATERMARK_CHARS.length();
        watermark.append(WATERMARK_CHARS.charAt(charIndex));
    }
    return watermark.toString();
}

// 添加水印到原始数据
public static String addWatermark(String input, long watermarkId) {
    String watermark = generateRandomWatermark(watermarkId);
    return input + watermark; // 直接拼接到原数据后面
}
```

水印溯源 把水印字符串转成10进制数 就是水印ID
```
// 提取水印ID
private static Pattern watermarkPattern = Pattern.compile("[\u200B\u200C\u200D]{16}");

public static Long extractWatermarkId(String watermarkedString) {
    Matcher matcher = watermarkPattern.matcher(watermarkedString);
    while (matcher.find()) {
        String group = matcher.group();
        long watermarkIdCandidate = parseWatermark(group);
        return watermarkIdCandidate;
    }
    return null;
}

private static long parseWatermark(String watermark) {
    long watermarkId = 0;
    for (int i = 0; i < watermark.length(); i++) {
        char watermarkChar = watermark.charAt(i);
        int charIndex = WATERMARK_CHARS.indexOf(watermarkChar);
        watermarkId += charIndex * Math.pow(WATERMARK_CHARS.length(), i);
    }
    return watermarkId;
}

```

链路水印 用户使用浏览器访问了服务x接口a 接口a里面又去调用了另外一个服务y的b接口 有调用了服务z接口c 调用下一个服务接口的时候会复用上一个服务接口的返回值作为入参 所以会带上水印id
维护水印id的时候顺便维护一个关系 这个关系是请求响应的时候建立的
最先c接口返回给y 经过代理 代理对结果的字段解析 抽取水印字符串 这时候是没有的 就建立一个水印id 放到结果里面 并且建立一个关系 这个关系是当前接口返回的应用id(服务z)和你客户端ip的应用id(服务y)
这样就建立了z-y的关系
然后到了b返回服务x 经过代理 代理对结果的字段解析 抽取水印字符串 这时候接过里面 并且建立一个关系 这个关系是当前接口返回的应用id(服务y)和你客户端ip的应用id(服务x)
这样就建立了y-x的关系
整条链路串起来了
