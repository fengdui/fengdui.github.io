---
title: "k近邻算法"
date: "2016-02-17"
tags: ["人工智能"]
ShowToc: false
TocOpen: false
---
K-近邻（KNN）：有监督学习，用于分类/回归（知道标签，预测新数据） 非线性 
k近邻算法说穿了就是对于每个要分类的用例，先和每一个样本求距离，然后找出距离最近的k个样本，这k个样本里面出现次数最多的类别就是这个样例的类别。每一个需要分类的用例都需要和每一个样本求距离，然后排序，求次数出现最多的类别，效率不高。

```python
import numpy as np
import operator
from os import listdir


def createDataSet():
    """创建示例数据集"""
    group = np.array([[1.0, 1.1], [1.0, 1.0], [0, 0], [0, 0.1]])
    labels = ['A', 'A', 'B', 'B']
    return group, labels


def classify0(inX, dataSet, labels, k):
    """k近邻分类器
    
    参数:
    inX -- 待分类的输入向量
    dataSet -- 训练数据集
    labels -- 训练数据集的标签
    k -- 近邻的数量
    
    返回:
    分类结果
    """
    dataSetSize = dataSet.shape[0]
    
    # 计算距离
    diffMat = np.tile(inX, (dataSetSize, 1)) - dataSet
    sqDiffMat = diffMat ** 2
    sqDistance = sqDiffMat.sum(axis=1)
    distances = sqDistance ** 0.5
    
    # 排序并获取索引
    sortedDistIndicies = distances.argsort()
    
    # 统计前k个近邻的标签
    classCount = {}
    for i in range(k):
        voteIlabel = labels[sortedDistIndicies[i]]
        classCount[voteIlabel] = classCount.get(voteIlabel, 0) + 1
    
    # 排序并返回出现次数最多的标签
    sortedClassCount = sorted(classCount.items(), 
                             key=operator.itemgetter(1), 
                             reverse=True)
    return sortedClassCount[0][0]


def file2matrix(filename):
    """将文本文件转换为矩阵"""
    with open(filename, 'r') as fr:
        arrayOfLines = fr.readlines()
        
    numbers = len(arrayOfLines)
    returnMat = np.zeros((numbers, 3))
    classLabelVector = []
    index = 0
    
    for line in arrayOfLines:
        line = line.strip()
        listFromLine = line.split('\t')
        returnMat[index, :] = listFromLine[0:3]
        classLabelVector.append(int(listFromLine[-1]))
        index += 1
    
    return returnMat, classLabelVector


def autoNorm(dataSet):
    """归一化数据集"""
    minVals = dataSet.min(0)
    maxVals = dataSet.max(0)
    ranges = maxVals - minVals
    
    m = dataSet.shape[0]
    normDataSet = (dataSet - np.tile(minVals, (m, 1))) / np.tile(ranges, (m, 1))
    
    return normDataSet, ranges, minVals


def datingClassTest():
    """测试约会网站数据集"""
    hoRatio = 0.10  # 测试集比例
    
    # 加载数据
    datingDataMat, datingLabels = file2matrix('datingTestSet2.txt')
    
    # 归一化
    normMat, ranges, minVals = autoNorm(datingDataMat)
    
    m = normMat.shape[0]
    numTestVecs = int(m * hoRatio)
    errorCount = 0.0
    
    # 测试分类器
    for i in range(numTestVecs):
        classifierResult = classify0(
            normMat[i, :], 
            normMat[numTestVecs:m, :], 
            datingLabels[numTestVecs:m], 
            3
        )
        
        print(f"分类结果: {classifierResult}, 实际结果: {datingLabels[i]}")
        
        if classifierResult != datingLabels[i]:
            errorCount += 1.0
    
    print(f"总错误率: {errorCount / float(numTestVecs)}")


def img2vector(filename):
    """将图像转换为向量"""
    returnVect = np.zeros((1, 1024))
    
    with open(filename, 'r') as fr:
        for i in range(32):
            lineStr = fr.readline()
            for j in range(32):
                returnVect[0, 32 * i + j] = int(lineStr[j])
    
    return returnVect


def handwritingClassTest():
    """测试手写数字数据集"""
    hwLabels = []
    
    # 加载训练数据
    trainingFileList = listdir('trainingDigits')
    m = len(trainingFileList)
    trainingMat = np.zeros((m, 1024))
    
    for i in range(m):
        fileNameStr = trainingFileList[i]
        fileStr = fileNameStr.split('.')[0]
        classNumStr = int(fileStr.split('_')[0])  # 修正了分隔符
        hwLabels.append(classNumStr)
        trainingMat[i, :] = img2vector(f'trainingDigits/{fileNameStr}')
    
    # 加载测试数据
    testFileList = listdir('testDigits')
    errorCount = 0.0
    mTest = len(testFileList)
    
    # 测试分类器
    for i in range(mTest):
        fileNameStr = testFileList[i]
        fileStr = fileNameStr.split('.')[0]
        classNumStr = int(fileStr.split('_')[0])  # 修正了分隔符
        vectorUnderTest = img2vector(f'testDigits/{fileNameStr}')
        
        classifierResult = classify0(vectorUnderTest, trainingMat, hwLabels, 3)
        print(f"分类结果: {classifierResult}, 实际结果: {classNumStr}")
        
        if classifierResult != classNumStr:
            errorCount += 1.0
    
    print(f"\n总错误数: {int(errorCount)}")
    print(f"总错误率: {errorCount / float(mTest)}")
```

```python
# 主函数调用
if __name__ == '__main__':
    import scipy
    
    # 1. 加载约会网站数据
    datingDataMat, datingLabels = file2matrix('datingTestSet2.txt')
    print("约会数据矩阵:")
    print(datingDataMat)
    print("\n约会数据标签:")
    print(datingLabels)
    
    # 2. 归一化数据
    normMat, ranges, minVals = autoNorm(datingDataMat)
    print("\n归一化数据:")
    print(normMat)
    
    # 3. 测试约会网站分类器
    print("\n约会网站分类器测试:")
    datingClassTest()
    
    # 4. 测试手写数字分类器
    print("\n手写数字分类器测试:")
    handwritingClassTest()
```