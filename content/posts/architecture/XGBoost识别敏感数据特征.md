---
title: "XGBoost识别敏感数据特征"
date: "2025-06-12"
tags: ["人工智能"]
ShowToc: false
TocOpen: false
---


### 特征工程 FeatureExtractUtil抽取特征

```java
public class FeatureExtractUtil {
    
    // 1. 长度统计特征
    public static void featureExtract1(FeatureContext context) {
        int maxLen = 0, minLen = Integer.MAX_VALUE, total = 0;
        for (FeatureValue value : context.getData()) {
            int len = value.length();
            maxLen = Math.max(maxLen, len);
            minLen = Math.min(minLen, len);
            total += len;
        }
        float avgLen = (float) total / context.getData().size();
        
        context.put(MAXIMUM_LENS, result(maxLen));
        context.put(MINIMUM_LENS, result(minLen));
        context.put(AVERAGE_LENS, result(avgLen));
        context.put(TOTAL_LENS, result(total));
    }
    
    // 2. 字符类型分布特征
    public static void featureExtract2(FeatureContext context) {
        Map<CharType, int[]> charTypeMap = getCharTypeMap();
        
        for (FeatureValue value : context.getData()) {
            for (CharType type : CharType.values()) {
                if (value.isOnlyType(type.getType())) {
                    charTypeMap.get(type)[0]++;
                }
                if (value.isType(type.getType())) {
                    charTypeMap.get(type)[1]++;
                }
            }
        }
        
        // 计算比例并存储特征
        context.put(PURE_NUM_RATIO, typeRatio(context, NUM));
        context.put(PURE_ENG_RATIO, typeRatio(context, ENG));
        context.put(CONTAIN_NUM_RATIO, typeRatio(context, NUM));
        // ...
    }
    
    // 3. 唯一值统计
    public static Float uniqueCount(FeatureContext context, int idx) {
        Set<Character> uniqueChars = context.getData().stream()
            .filter(v -> v.length() > idx)
            .map(v -> v.charAt(idx))
            .collect(toSet());
        return result(uniqueChars.size());
    }
    
    // 4. 字符类型比例计算
    public static Float typeRatio(FeatureContext context, CharType type) {
        long count = context.getData().stream()
            .mapToInt(v -> v.getType(type) ? 1 : 0)
            .sum();
        return ratio(count, context.getData().size());
    }
}
```

### 特征向量生成

```java
// 100+维特征向量示例
float[] features = {
    12.0f,      // 最大长度
    8.0f,       // 最小长度  
    10.5f,      // 平均长度
    0.83f,      // 英文字符比例
    0.17f,      // 数字字符比例
    0.95f,      // 语义相似度
    1.0f,       // 包含敏感词
    // ... 共100+维
};
int label = 1;  // 1=敏感, 0=非敏感
```

### 模型训练实现

```java
@Component
public class FeatureModesGenerate {
    
    @Autowired
    private FeatureModelComponent featureModelComponent;
    
    @Autowired 
    private FeatureGenerateResultMapper featureGenerateResultMapper;
    
    public void buildModel(Integer industryId) throws Exception {
        // 1. 查询特征数据
        List<Long> bizIds = queryFeatureBizIds(industryId);
        if (CollectionUtils.isEmpty(bizIds)) {
            log.warn("没有特征数据，无法生成模型");
            return;
        }
        
        // 2. 准备文件路径
        String tempPath = TEMP_DIR + File.separator + UUID.randomUUID();
        String modelPath = featureModelComponent.getModelPath(tempPath, industryId);
        String trainPath = featureModelComponent.getTrainPath(tempPath, industryId);
        String testPath = featureModelComponent.getTestPath(tempPath, industryId);
        String bizPath = featureModelComponent.getBizPath(tempPath, industryId);
        
        // 3. 生成训练数据
        Map<Long, Integer> bizIndexMap = new LinkedHashMap<>();
        try (BufferedWriter trainWriter = new BufferedWriter(new FileWriter(trainPath));
             BufferedWriter testWriter = new BufferedWriter(new FileWriter(testPath));
             BufferedWriter bizWriter = new BufferedWriter(new FileWriter(bizPath))) {
            
            int index = 0;
            int testSize = computeTestNum(bizIds.size());
            
            for (Long bizId : bizIds) {
                // 查询特征数据
                LambdaQueryWrapper<FeatureGenerateResultDO> q = Wrappers.lambdaQuery(FeatureGenerateResultDO.class)
                    .eq(FeatureGenerateResultDO::getIndustryId, industryId)
                    .eq(FeatureGenerateResultDO::getBizId, bizId);
                    
                List<FeatureGenerateResultDO> features = featureGenerateResultMapper.selectList(q);
                
                // 分配到训练集或测试集
                if (index < testSize) {
                    writeFeature(industryId, features, testWriter);
                } else {
                    writeFeature(industryId, features, trainWriter);
                }
                
                // 写入业务数据
                writeFeature(industryId, features, bizWriter);
                bizIndexMap.put(bizId, index++);
            }
        }
        
        // 4. 调用XGBoost训练
        if (bizIds.size() < 10) {
            log.warn("样本数量不够，无法生成模型");
            return;
        }
        
        FeatureUtils.trainModel(trainPath, testPath, modelPath, industryId);
        
        // 5. 清理临时文件
        featureModelComponent.deleteModel(industryId);
        transfer(modelPath, featureModelComponent.getModelPath(industryId));
    }
    
    // 写入特征数据
    private void writeFeature(Integer label, List<FeatureGenerateResultDO> features, 
                             BufferedWriter writer) throws IOException {
        String featureStr = FeatureUtils.ofFeatureLabelStr(label, features);
        writer.write(featureStr);
        writer.newLine();
    }
    
    // 计算测试集大小
    private int computeTestNum(int totalNum) {
        return (int) Math.floor(totalNum * 0.3);  // 30%作为测试集
    }
}
```

### XGBoost训练调用

```java
public class FeatureUtils {
    
    // 调用XGBoost训练
    public static void trainModel(String trainPath, String testPath, 
                                 String modelPath, int industryId) {
        try {
            // 1. 加载训练数据
            DMatrix trainMatrix = new DMatrix(trainPath);
            DMatrix testMatrix = new DMatrix(testPath);
            
            // 2. 设置XGBoost参数
            Map<String, Object> params = new HashMap<>();
            params.put("objective", "binary:logistic");  // 二分类
            params.put("eval_metric", "logloss");
            params.put("max_depth", 6);
            params.put("eta", 0.1);
            params.put("subsample", 0.8);
            params.put("colsample_bytree", 0.8);
            
            // 3. 训练模型
            Map<String, DMatrix> watches = new HashMap<>();
            watches.put("train", trainMatrix);
            watches.put("test", testMatrix);
            
            Booster booster = XGBoost.train(trainMatrix, params, 100, watches, null, null);
            
            // 4. 保存模型
            booster.saveModel(modelPath);
            
        } catch (XGBoostError e) {
            throw new RuntimeException("XGBoost训练失败", e);
        }
    }
}
```

### 实际预测调用

```java
// 字段预测示例
public class SensitiveDataClassifier {
    
    public PredictionResult predict(String columnName, String columnComment) {
        // 1. 特征提取
        FeatureContext context = new FeatureContext();
        context.addData(new FeatureValue(columnName));
        
        // 提取所有特征
        FeatureExtractUtil.featureExtract1(context);
        FeatureExtractUtil.featureExtract2(context);
        // ... 其他特征提取
        
        // 2. 构建特征向量
        float[] features = context.toFeatureVector();
        
        // 3. XGBoost预测
        try {
            Booster booster = XGBoost.loadModel(modelPath);
            DMatrix dMatrix = new DMatrix(features, 1, features.length);
            float[][] predictions = booster.predict(dMatrix);
            
            float probability = predictions[0][0];
            boolean isSensitive = probability > 0.5f;
            
            return new PredictionResult(isSensitive, probability);
            
        } catch (XGBoostError e) {
            throw new RuntimeException("预测失败", e);
        }
    }
}

// 预测结果
class PredictionResult {
    private boolean sensitive;
    private float confidence;
    
    // getters...
}
```

### 字段特征提取示例

```java
// 字段: "phone_number"
FeatureContext context = extractFeatures("phone_number");

// 生成的特征向量 (部分)
float[] features = {
    12.0f,          // 字段长度
    0.83f,          // 英文字符比例
    0.17f,          // 下划线比例  
    0.0f,           // 数字比例
    2.0f,           // 唯一词汇数
    0.95f,          // 语义相似度
    1.0f,           // 包含敏感词
    0.75f,          // 字符复杂度
    // ... 共100+维
};

// 训练数据格式 (LibSVM)
// 1 1:12.0 2:0.83 3:0.17 4:0.0 5:2.0 6:0.95 7:1.0 8:0.75 ...
```

### 复杂字段处理

```java
// 测试用例
Map<String, Float> testCases = Map.of(
    "usr_mobile_info_bak", 0.92f,      // 92%概率敏感
    "customer_id_card_hash", 0.88f,    // 88%概率敏感
    "encrypted_phone_data", 0.94f,     // 94%概率敏感
    "user_login_time", 0.15f,          // 15%概率敏感
    "system_config_flag", 0.08f        // 8%概率敏感
);

// 批量预测
for (Map.Entry<String, Float> entry : testCases.entrySet()) {
    PredictionResult result = classifier.predict(entry.getKey(), "");
    System.out.printf("%s: %.2f (expected: %.2f)\n", 
        entry.getKey(), result.getConfidence(), entry.getValue());
}
```


### Top3评估实现

```java
public class Top3Evaluation {
    
    public static class IndexScore {
        private int index;
        private double score;
        
        // constructors, getters...
    }
    
    public static double evaluateTop3(List<IndexScore> predictions, int actualIndex) {
        // 按分数降序排序
        predictions.sort((a, b) -> Double.compare(b.score, a.score));
        
        // 检查Top3中是否包含正确答案
        for (int i = 0; i < Math.min(3, predictions.size()); i++) {
            if (predictions.get(i).index == actualIndex) {
                return 1.0 / (i + 1);  // 位置越靠前分数越高
            }
        }
        return 0.0;
    }
}
```