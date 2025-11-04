---
title: "ac自动机结合trie树实现敏感词发现"
date: "2025-03-31"
tags: ["架构"]
ShowToc: false
TocOpen: false
---




###  句子切割成词 使用ac自动机匹配转成中文 在合并词作为句子 进行字典树匹配
**场景**：识别`phoneNumber`、`customerIdCard` 等包含敏感词的复合字段名对应的中文
比如customerIdCard->customer id card -> 客户身份证号
然后拿着客户身份证号 匹配trie


```
用户的的信息可能不是标准的 直接trie树不一定匹配成功 所以要将字段名使用翻译引擎转换之后在调用trie树
## 智能翻译引擎  `usernameInfo` → `用户名信息`
先切割usernameInfo成username info
识别大写字母边界 识别数字边界 分割成单词列表
```java
// 翻译主流程
@Override
public String translate(String name) {
    List<String> tokens = split(name);  // 智能分词
    
    return tokens.stream()
                 .map(this::translateFast)      // 词典查找
                 .filter(Objects::nonNull)      // 过滤空值
                 .collect(joining());           // 拼接结果
}

// 智能分词算法
public List<String> split(CharSequence str) {
    List<String> result = new ArrayList<>();
    StringBuilder sb = new StringBuilder();
    
    byte prevType = 0;  // 前一字符类型
    
    for (int i = 0; i < str.length(); i++) {
        char c = str.charAt(i);
        
        // 字符类型判断
        boolean isUpper = Character.isUpperCase(c);
        boolean isLower = Character.isLowerCase(c);
        boolean isNumber = CharUtil.isNumber(c);
        
        // 分词边界判断
        if (shouldSplit(isUpper, isLower, isNumber, prevType)) {
            if (sb.length() > 0) {
                result.add(sb.toString());
                sb.setLength(0);
            }
        }
        
        sb.append(Character.toLowerCase(c));
        prevType = getCharType(isUpper, isLower, isNumber);
    }
    
    return result;
}
```
对于上面的切割只是简单的驼峰这些 可能切割之后还是多个单词的 比如用户输入的是usernameInfo 上面切割成了username和info
对上一步的每个单词继续 使用MinNumSplit
复合词的最优切分 使用了ac自动机 匹配到了之后转成中文
MinNumSplit 解决了`username` vs `user` + `name` 的选择问题
```
输入: "username"
词典: ["user->用户", "name->名称", "username->用户名"]
结果是用户名(匹配到了username) 
对于info也是同样的操作得到中文信息

```

```java
// AC自动机 + 动态规划
@Override
public List<String> split(String text) {
    char[] chars = text.toCharArray();
    Node[] dp = new Node[chars.length];  // DP状态数组
    
    TrieNode current = getRoot();
    
    // AC自动机扫描
    for (int i = 0; i < chars.length; i++) {
        char c = chars[i];
        
        // 状态转移
        TrieNode next = current.getChildByName(c);
        while (next == null && current != root) {
            current = current.getFailure();  // 失败函数
            next = current.getChildByName(c);
        }
        current = next ?? root;
        
        // 处理匹配结果
        List<ValueLenResult> matches = current.getResults();
        for (ValueLenResult match : matches) {
            int len = match.getLen();
            int start = i - len + 1;
            
            // DP状态更新 - 最小分词数
            updateDP(dp, start, i, match);
        }
    }
    
    // 回溯构建最优解
    return backtrack(dp, chars.length - 1);
}

// DP状态更新
private void updateDP(Node[] dp, int start, int end, ValueLenResult match) {
    Node prev = (start > 0) ? dp[start - 1] : null;
    int newCost = (prev != null) ? prev.getNum() + 1 : 1;
    
    // 贪心选择：最小分词数
    if (dp[end] == null || newCost < dp[end].getNum()) {
        dp[end] = new Node(newCost, start - 1, match.getV());
    }
}
```

// Trie树匹配核心
private List<TargetResult> autoMatch(String text, HitType hitType) {
List<TargetInfo> matches = match.find(text.trim());  // O(m) 时间复杂度

    return matches.stream()
                  .map(info -> new TargetResult(hitType, info))
                  .collect(toList());
}

最后得到转换后的信息 用户名+信息 合起来得到用户名信息匹配字典树
```java
// 核心数据结构：Trie树
private TrieMatch<TargetInfo> match;
private TranslateEngine translateEngine;

@Override
public void execute(EngineContext context) {
    // 1. 表注释匹配
    String tableComment = context.getTable().getTableComment();
    List<TargetResult> results = autoMatch(tableComment, HitType.COMMENT_SIMILAR);
    
    // 2. 字段级处理
    context.getColumns().forEach(column -> {
        NyxMetaColumn meta = column.getMeta();
        
        // 2.1 翻译后匹配（中文）- 关键创新点
        String translatedName = translateEngine.translate(columnName);
        if (!isEmpty(translatedName)) {
            results.addAll(autoMatch(translatedName, HitType.NAME_SIMILAR));
        }
        
        context.addAllTarget(results);
    });
}