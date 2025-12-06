---
title: "venv和conda"
date: "2025-12-06"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
开始做python项目，需要使用虚拟环境 学习conda和venv  
venv项目结构
```
your-project/
├── venv/                 # 虚拟环境（在 .gitignore 中）
├── src/                  # 源代码
│   ├── __init__.py
│   └── main.py
├── tests/                # 测试代码
├── requirements.txt      # 依赖列表
├── .gitignore           # 忽略 venv
└── README.md
```
source venv/bin/activate  # Linux/Mac  
venv\Scripts\activate     # Windows
python -m pip install --upgrade pip  

conda
```

conda remove -n voiceprint-api --all -y
conda create -n voiceprint-api python=3.10 -y
conda activate voiceprint-api
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/
pip config set global.index-url  https://pypi.tuna.tsinghua.edu.cn/simple
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main 
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free 
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge
pip install -r requirements.txt
```