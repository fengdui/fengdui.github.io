---
title: "nova opensearch knn llm图片查询实战"
date: "2026-01-07"
tags: ["人工智能"]
ShowToc: false
TocOpen: false
---
OpenSearch的KNN（K最近邻）向量搜索，是通过KNN插件实现的一种高级搜索方式。它让你可以将文本、图片等内容转化为多维向量并存入索引，然后通过搜索与查询向量最相似的向量来返回结果，非常适用于语义搜索、相似图片查找等场景  

# 写入流程
先使用摄像头进行抓拍 或者视频进行抽帧  
使用多模态大模型处理  Amazon Mmc Nova Embedding  透出一个lable和一段描述  
使用 Amazon GCR Nova Embedding 生成文本的1024维向量嵌入（embedding） 写入opensearch  
然后就可以根据描述查询最相似的图片  
# 查询
用户输入一段话  
使用 Amazon GCR Nova Embedding 将非英文文本翻译成英文  
同样使用 Amazon GCR Nova Embedding 对翻译后对文本转成1024维向量  
KNN 查询 在 aws_gcr_nova_description_embedding 字段上进行 KNN 搜索 k = pageSize * 2，返回更多候选以便后续 rerank  
分数权重调整 使用 script_score 将向量相似度分数乘以配置的权重（如 0.7） 公式：最终分数 = 向量相似度分数 × embeddingScore  
添加到查询 使用 should（OR），与关键词匹配组合  
构建 match 查询 在 description 字段上进行全文匹配 分数权重调整 同样使用 script_score 调整分数 公式：最终分数 = 关键词匹配分数 × descScore 步骤3：添加到查询 同样使用 should，与向量搜索组合  
混合搜索逻辑 最终查询 = filter条件 AND (向量搜索 OR 关键词匹配) filter：必须满足（cid、uid、时间等） should：至少满足一个（向量搜索或关键词匹配） 最终分数：取两个 should 子句中的最高分  
NN 查询返回：pageSize * 2 个最相似的文档 例如：pageSize = 10，则返回 20 个向量相似度最高的文档  
关键词匹配  Match查询返回所有匹配的文档（无限制）  OpenSearch 合并两个结果集，按分数排序  应用size限制 返回 pageSize + 1 个结果  
应用层处理 进行rerank 调用AWS Bedrock Nova模型 最终返回给用户pageSize 个结果  
得到最相似的图片描述的事件 再去查询得到图片  
```
"aws_gcr_nova_description_embedding": {
  "method": {
    "engine": "lucene",
    "space_type": "cosinesimil",
    "name": "hnsw",
    "parameters": {
      "ef_construction": 256,
      "m": 32
    }
  },
  "type": "knn_vector",
  "dimension": 1024
}
```
rerank的提示词
```
spring.os.rerank-server-properties.system-description-text=You are a semantic document rater that evaluates relevance based on conceptual meaning rather than keyword matching. <core_principle> Score documents by semantic relationships: direct concepts, categorical relationships, functional associations, and contextual connections. Focus on what concepts mean, not just word matches. </core_principle> <scoring_framework> 1.0-0.8: Direct semantic match (synonyms, exact concepts) 0.8-0.6: Categorical relationship (type-instance, part-whole) 0.6-0.4: Functional/contextual association (related activities, environments) 0.4-0.2: Weak conceptual connection (thematic similarity) 0.2-0.0: No meaningful relationship </scoring_framework> <semantic_examples> Query "activity": "person walking" \u2192 0.8 (walking is an activity) Query "interaction": "people talking" \u2192 0.9 (talking is interaction) Query "professional": "person in suit" \u2192 0.7 (suit suggests professional context) Query "movement": "door opening" \u2192 0.6 (opening involves movement) </semantic_examples> <output_requirements> - Score ALL documents in exact input order - Output exactly DOCUMENT_COUNT JSON objects - Include every index 0 to {DOCUMENT_COUNT-1} - No skipping or filtering documents </output_requirements>
spring.os.rerank-server-properties.user-description-text=Rate documents for semantic relevance to: ${input} Apply semantic understanding: - Consider conceptual meaning beyond word matching - Evaluate functional and contextual relationships - Score based on semantic connection strength Rate the relevance of this document for the given query. Query: Cleaning the lobby Documents: ${document} Provide scores for each aspect and a total score out of 1. <output_format> [{"i": 0, "s": score}..}] </output_format> Important: -Each item must be evaluated and scored, ensuring no element is overlooked or omitted. Skip the preamble and start directly with the answer, please follow <output_format>, just output total score.
```