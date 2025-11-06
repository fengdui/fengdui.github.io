---
title: "使用bert模型进行敏感词分析"
date: "2025-05-23"
tags: ["人工智能"]
ShowToc: false
TocOpen: false
---

bert模型是基于Transformer架构的双向预训练语言模型
这里得到bert模型转成onnx格式 onnx可以在java里面运行 无需Python环境 比原始框架更快的推理速度 更小的内存占用
```
public VectorConvEngine createSimilarityEngine() {
   if (StrUtil.isEmpty(this.config.getOnnx())) {
      return null;
   } else {
      OrtEnvironment env = OrtEnvironment.getEnvironment();
   
      OrtSession session;
      try {
          session = env.createSession(this.config.getOnnx());
      } catch (OrtException e) {
          log.warn("create session:{}", e.getMessage());
          return null;
      }
   
      TransformerTokenizer tokenizer = this.getTokenizer();
      return new PredictorEngine(env, session, tokenizer, this.getVectorCache());
   }
}
```

使用TransformerTokenizer转成BertToken 里面用到了vocab.txt和WordpieceTokenizer BasicTokenizer
"user_phone_number"
↓
1. 基础分词: ["user", "phone", "number"]
   ↓
2. WordPiece分词: ["user", "phone", "num", "##ber"]
   ↓
3. 添加特殊标记: ["[CLS]", "user", "phone", "num", "##ber", "[SEP]"]
   ↓
4. 转换为ID: [101, 2025, 3042, 3841, 2121, 102]
   ↓
5. 创建掩码: [1, 1, 1, 1, 1, 1, 0, 0, ..., 0]
   ↓
6. 输出BertToken对象


调用bert模型得到向量
```
public float[] getTensor(String text) throws OrtException {
     BertToken bertToken = this.tokenizer.sentence2Ids(text);
     if (null == bertToken) {
         return null;
     } else {
         long[][] inputIds = new long[1][];
         inputIds[0] = bertToken.getInput_ids();
         long[][] attentionMask = new long[1][];
         attentionMask[0] = bertToken.getAttention_mask();
         OnnxTensor a = OnnxTensor.createTensor(this.env, inputIds);
         OnnxTensor b = OnnxTensor.createTensor(this.env, attentionMask);
         Map<String, OnnxTensor> inputMap = new HashMap(2);
         Set<String> requestedOutputs = new HashSet(1);
         inputMap.put("input_ids", a);
         inputMap.put("attention_mask", b);
         requestedOutputs.add("output_0");

         float[] var11;
         try (OrtSession.Result r = this.session.run(inputMap, requestedOutputs)) {
             var11 = this.unwrap(r);
         }

         return var11;
     }
}
private float[] unwrap(OrtSession.Result result) throws OrtException {
    try {
        // 获取模型输出张量
        OnnxValue outputValue = result.get(0);
        
        // 转换为三维浮点数组 [batch_size, sequence_length, hidden_size]
        float[][][] output = (float[][][]) outputValue.getValue();
        
        // 返回第一个样本的向量表示（通常取[CLS]位置的向量）
        return output[0][0]; // [CLS] token的向量
        
    } catch (OrtException | Exception e) {
        throw new RuleException("ONNX推理失败", e);
    }
}

 
```