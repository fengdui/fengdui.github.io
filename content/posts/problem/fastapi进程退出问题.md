---
title: "fastapi进程退出问题"
date: "2025-12-13"
tags: ["问题"]
ShowToc: false
TocOpen: false
---
使用了strace dmsg等命令发现一切正常  
使用了各种监控cpu内存使用的脚本都没发现问题  
代码里面注册了很多信号处理 正常退出能够触发这个回调  
```
signal.signal(signal.SIGINT, signal_handler)
signal.signal(signal.SIGTERM, signal_handler)
signal.signal(signal.SIGSEGV, signal_handler)
signal.signal(signal.SIGABRT, signal_handler)
signal.signal(signal.SIGILL, signal_handler)
def signal_handler(signum, frame):
    """信号处理器"""
    logger.info(f"收到信号 {signum}，正在优雅关闭服务...")
    sys.exit(0)
```
说明不是系统触发杀死进程的  
uvicorn启动的时候有个参数limit_max_requests  
```
uvicorn.run(
    "app.application:app",
    host=settings.host,
    port=settings.port,
    reload=False,  # 生产环境关闭热重载
    workers=1,  # 单进程模式，避免模型重复加载
    access_log=False,  # 关闭uvicorn自带access日志
    log_level="info",
    timeout_keep_alive=30,  # keep-alive超时
    timeout_graceful_shutdown=300,  # 优雅关闭超时
    limit_concurrency=1000,  # 并发连接限制
    limit_max_requests=1000,  # 最大请求数限制
    backlog=2048,  # 连接队列大小
)
```