---
title: "通过jpmml来实现机器学习模型的加载和评估"
date: "2018-02-24"
tags: ["人工智能"]
ShowToc: false
TocOpen: false
---

通过org.jpmml.evaluator.ModelEvaluator接口来实现机器学习模型的加载和评估。以下是项目中JPMML的具体使用方式：

JPMML模型加载器 将模型维护到一个map里面
并且将怎么获取特征(作为一个函数)维护到map里面
当数据过来的时候调用这个函数去获取特征 这是一个命令模式
每个模型怎么提取每个特征模型使用者是不知道的 是发布模型的人定义的怎么得到特征的一个函数 这个函数接收一个对象 返回一个值
这个函数得到的是一段 加载模型的时候同时使用Janino去加载这个java代码 

```
import org.jpmml.evaluator.ModelEvaluator;
import org.springframework.http.ResponseEntity;
import org.springframework.http.HttpStatus;
import org.springframework.web.client.RestTemplate;
import com.alibaba.fastjson.JSONObject;
import com.alibaba.fastjson.JSONArray;
import org.apache.commons.codec.binary.Base64;
import org.apache.commons.io.SerializationUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.util.*;

/**
 * JPMML模型加载器
 * 负责从远程服务器获取、解析和管理JPMML模型
 */
public class JpmmlModelLoader implements Runnable {
    private static final Logger logger = LoggerFactory.getLogger(JpmmlModelLoader.class);
    
    // 注入的依赖
    private final RestTemplate restTemplate;
    private final BasicModelInfo modelInfoStore;
    
    // 模型版本管理映射
    private final Map<Integer, Long> modelVersionMap = new HashMap<>();
    
    // 模型配置包列表
    private final List<JaninoModelScriptPackageConfig> modelPackageConfigs = new ArrayList<>();
    
    /**
     * 构造函数
     * @param restTemplate HTTP请求模板
     * @param modelInfoStore 模型信息存储器
     */
    public JpmmlModelLoader(RestTemplate restTemplate, BasicModelInfo modelInfoStore) {
        this.restTemplate = restTemplate;
        this.modelInfoStore = modelInfoStore;
    }
    
    /**
     * 重新加载指定URL的模型
     * @param modelUrl 模型URL地址
     */
    public void reloadModel(String modelUrl) {
        try {
            logger.debug("开始重新加载模型: {}", modelUrl);
            
            // 发送HTTP请求获取模型数据
            ResponseEntity<JSONObject> response = restTemplate.getForEntity(modelUrl, JSONObject.class);
            HttpStatus statusCode = response.getStatusCode();
            
            // 检查响应状态
            if (statusCode == HttpStatus.OK) {
                JSONObject responseBody = response.getBody();
                processModelResponse(responseBody);
            } else {
                logger.warn("模型URL返回空数据: {}", modelUrl);
            }
        } catch (Exception e) {
            logger.error("执行HTTP请求出错", e);
        }
    }
    
    /**
     * 处理模型响应数据
     * @param responseBody 模型响应JSON对象
     */
    private void processModelResponse(JSONObject responseBody) {
        // 获取模型存储Map
        Map<String, List<Object>> paramInfoMap = modelInfoStore.getParamInfoMaps();
        Map<String, ModelEvaluator<?>> modelMap = modelInfoStore.getModelFileMaps();
        
        // 提取数据数组
        JSONArray dataArray = responseBody.getJSONArray("data");
        List<String> activeModelNames = new ArrayList<>();
        
        // 处理每个模型数据
        for (Object item : dataArray) {
            JSONObject modelData = (JSONObject) item;
            Integer modelId = modelData.getInteger("id");
            String modelName = modelData.getString("modelName");
            activeModelNames.add(modelName);
            
            // 获取模型更新时间
            Long updateTime = (modelData.getDate("updateTime") == null) ? -1L : modelData.getDate("updateTime").getTime();
            Long lastUpdateTime = (modelVersionMap.get(modelId) == null) ? -1L : modelVersionMap.get(modelId);
            
            // 检查是否需要更新模型
            if (updateTime > lastUpdateTime) {
                try {
                    updateModel(modelData, modelId, modelName, paramInfoMap, modelMap, updateTime);
                } catch (IOException e) {
                    logger.error("更新模型发生异常.", e);
                }
            }
        }
        
        // 处理已删除的模型
        removeUnusedModels(activeModelNames, paramInfoMap, modelMap);
    }
    
    /**
     * 更新模型数据
     */
    private void updateModel(JSONObject modelData, Integer modelId, String modelName, 
                            Map<String, List<Object>> paramInfoMap, 
                            Map<String, ModelEvaluator<?>> modelMap, 
                            Long updateTime) throws IOException {
        // 解析在线特征配置
        String onlineFeatureBlobs = modelData.getString("onlineFeatureBlobs");
        byte[] featureBytes = Base64.decodeBase64(onlineFeatureBlobs);
        JSONObject featureJson = JSON.parseObject(new String(featureBytes));
        JSONArray jarsArray = featureJson.getJSONArray("jars");
        
        // 编译模型相关代码
        List<Object> paramList = JaninoCookCode.cookcode(jarsArray, featureJson);
        
        // 核心JPMML模型加载部分
        byte[] modelBytes = modelData.getBytes("modelInstance");
        ModelEvaluator<?> modelEvaluator = (ModelEvaluator<?>) SerializationUtils.deserialize(modelBytes);
        
        // 存储模型和参数信息
        paramInfoMap.put(modelName, paramList);
        modelMap.put(modelName, modelEvaluator);
        modelVersionMap.put(modelId, updateTime);
        
        logger.info("模型已更新 - ID: {}, 名称: {}", modelId, modelName);
        logger.debug("更新内容: {}", featureJson);
    }
    
}        
    
    
    

/**
 * 模型信息存储类
 * 用于存储模型参数和模型对象
 */
class BasicModelInfo {
    private final Map<String, List<Object>> paramInfoMaps = new HashMap<>();
    private final Map<String, ModelEvaluator<?>> modelFileMaps = new HashMap<>();
    
    public Map<String, List<Object>> getParamInfoMaps() {
        return paramInfoMaps;
    }
    
    public Map<String, ModelEvaluator<?>> getModelFileMaps() {
        return modelFileMaps;
    }
}

/**
 * 模型脚本包配置类
 * 包含模型包URL和版本信息
 */
class JaninoModelScriptPackageConfig {
    private final Map<String, Long> urlMap = new HashMap<>();
    
    public Map<String, Long> getUrlMap() {
        return urlMap;
    }
}

/**
 * Janino代码编译工具类
 */
class JaninoCookCode {
    /**
     * 编译模型相关代码
     */
    public static List<Object> cookcode(JSONArray jarsArray, JSONObject featureJson) {
        // 实际实现中会编译模型所需的Java代码
        // 这里返回空列表作为示例
        return new ArrayList<>();
    }
}
```
模型使用流程的推断实现 需要先获取数据 对数据调用每个特征对应的函数获取特征值 在调用模型
```
// 模型使用流程的推断实现
public class ModelService {
    private BasicModelInfo basicModelInfo; // 包含已加载的模型
    
    // 获取模型并执行预测
    public Map<String, Object> evaluateModel(String modelName, Map<String, Object> inputData) {
        // 1. 从内存中获取预加载的模型
        ModelEvaluator<?> modelEvaluator = basicModelInfo.getModelFileMaps().get(modelName);
        
        if (modelEvaluator == null) {
            throw new RuntimeException("模型" + modelName + "未加载");
        }
        
        try {
            // 2. 准备模型输入数据
            Map<String, FieldValue> arguments = new LinkedHashMap<>();
            
            // 将业务数据转换为JPMML需要的FieldValue格式
            for (Map.Entry<String, Object> entry : inputData.entrySet()) {
                String fieldName = entry.getKey();
                Object value = entry.getValue();
                
                FieldName fieldKey = FieldName.create(fieldName);
                Field<?> field = modelEvaluator.getInputFields().get(fieldKey);
                
                if (field != null) {
                    FieldValue fieldValue = field.prepare(value);
                    arguments.put(fieldName, fieldValue);
                }
            }
            
            // 3. 执行模型预测
            Map<FieldName, FieldValue> results = modelEvaluator.evaluate(arguments);
            
            // 4. 处理预测结果
            Map<String, Object> outputData = new LinkedHashMap<>();
            for (Map.Entry<FieldName, FieldValue> entry : results.entrySet()) {
                String fieldName = entry.getKey().getValue();
                FieldValue fieldValue = entry.getValue();
                
                // 提取结果值
                Object value = fieldValue.getValue();
                outputData.put(fieldName, value);
            }
            
            return outputData;
        } catch (Exception e) {
            throw new RuntimeException("模型评估失败: " + e.getMessage(), e);
        }
    }
}
```