---
title: "uvicorn框架日志loguru不打印到文件"
date: "2025-11-03"
tags: ["问题"]
ShowToc: false
TocOpen: false
---


uvicorn框架启动有日志 但是只在控制台有 
```
message = "Started server process [%d]"
color_message = "Started server process [" + click.style("%d", fg="cyan") + "]"
logger.info(message, process_id, extra={"color_message": color_message})
```

uvicorn框架里面的日志配置 位于框架里面的config.py
```
LOGGING_CONFIG: dict[str, Any] = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "default": {
            "()": "uvicorn.logging.DefaultFormatter",
            "fmt": "%(levelprefix)s %(message)s",
            "use_colors": None,
        },
        "access": {
            "()": "uvicorn.logging.AccessFormatter",
            "fmt": '%(levelprefix)s %(client_addr)s - "%(request_line)s" %(status_code)s',  # noqa: E501
        },
    },
    "handlers": {
        "default": {
            "formatter": "default",
            "class": "logging.StreamHandler",
            "stream": "ext://sys.stderr",
        },
        "access": {
            "formatter": "access",
            "class": "logging.StreamHandler",
            "stream": "ext://sys.stdout",
        },
    },
    "loggers": {
        "uvicorn": {"handlers": ["default"], "level": "INFO", "propagate": False},
        "uvicorn.error": {"level": "INFO"},
        "uvicorn.access": {"handlers": ["access"], "level": "INFO", "propagate": False},
    },
}

logger = logging.getLogger("uvicorn.error")
```
默认是输出到sys.stderr
然后现在项目是用了loguru 项目写了一个LoggingHandler去转发logging模块的日志 但是没有转发stderr和stdout
LoggingHandler的定义如下
```
class LoggingHandler(logging.Handler):
    """拦截logging日志并转发到loguru"""

    def emit(self, record):
        # 获取对应的loguru level
        try:
            level = logger.level(record.levelname).name
        except ValueError:
            level = record.levelno
    
        # 获取logger名称
        logger_name = record.name
        if logger_name.startswith("uvicorn"):
            logger_name = "uvicorn"
        elif logger_name.startswith("fastapi"):
            logger_name = "fastapi"
        elif logger_name.startswith("modelscope"):
            logger_name = "modelscope"
        elif logger_name.startswith("torch"):
            logger_name = "torch"
        elif logger_name.startswith("pydantic"):
            logger_name = "pydantic"
        elif logger_name.startswith("app."):
            logger_name = record.name
    
        # 处理uvicorn的color_message
        message = record.getMessage()
        if hasattr(record, 'extra') and 'color_message' in record.extra:
            message = record.extra['color_message'] % record.args
    
        # 转发到loguru
        logger.opt(exception=record.exc_info).bind(
            name=logger_name, version=VERSION
        ).log(level, message)
```
LoggingHandler的使用如下 拦截logging日志并转发到loguru 使用LoggingHandler替换logging的handler
```
def setup_logging(level: Optional[str] = None) -> None:
    """
    设置应用日志配置，使用loguru实现优雅的分段颜色显示

    格式: 时间[青色] 版本号[蓝色] 模块[灰色]-级别[彩色]-消息[绿色]
    示例: 250705 13:33:23[0.6.2][core.utils.modules_initialize]-INFO-初始化组件: intent成功

    Args:
        level: 日志级别
    """
    # 获取配置
    log_level = level or settings.logging.get("level", "INFO")

    # 确保日志目录存在
    log_dir = "logs"
    os.makedirs(log_dir, exist_ok=True)

    # 控制台输出格式 - 分段颜色显示
    console_format = (
        "<cyan>{time:YYMMDD HH:mm:ss}</cyan>"
        "<blue>[{extra[version]}]</blue>"
        "<light-black>[{name}]</light-black>-"
        "<level>{level}</level>-"
        "<green>{message}</green>"
    )

    # 文件输出格式 - 无颜色，保持相同格式
    file_format = (
        "{time:YYMMDD HH:mm:ss}" "[{extra[version]}]" "[{name}]-" "{level}-" "{message}"
    )

    # 添加控制台处理器
    logger.add(
        sys.stdout,
        format=console_format,
        level=log_level,
        colorize=True,
        backtrace=True,
        diagnose=True,
        enqueue=True,
    )

    # 添加文件处理器
    logger.add(
        os.path.join(log_dir, "voiceprint_api.log"),
        format=file_format,
        level=log_level,
        rotation="10 MB",
        retention="7 days",
        compression="gz",
        encoding="utf-8",
        backtrace=True,
        diagnose=True,
        enqueue=True,
    )

    # 拦截所有logging日志
    # 1. 移除root logger的所有handler
    for handler in logging.root.handlers[:]:
        logging.root.removeHandler(handler)

    # 2. 设置root logger只使用我们的handler
    logging.basicConfig(handlers=[LoggingHandler()], level=0, force=True)

    # 3. 强制替换所有已存在的logger的handler
    intercept_handler = LoggingHandler()
    for name in logging.root.manager.loggerDict:
        log = logging.getLogger(name)
        # 移除所有现有handler
        for handler in log.handlers[:]:
            log.removeHandler(handler)
        # 添加我们的handler
        log.addHandler(intercept_handler)
        # 设置propagate为False，防止重复输出
        log.propagate = False

    # 设置第三方库的日志级别
    logger.bind(version=VERSION).info(f"日志系统初始化完成，级别: {log_level}")
```
设置日志 替换LoggingHandler 但是这时候还没有uvicorn的logger 所以uvicorn还是走了之前的的  
uvicorn之前也是默认走到标准输出 那么这里把标准输出也给捕获到loguru  
替换标准流以捕获直接写入sys.stderr和sys.stdout的日志  
```
sys.stderr = StderrHandler(sys.stderr)
sys.stdout = StderrHandler(sys.stdout)
```
class StderrHandler:
```
class StderrHandler:
    
    def __init__(self, original_stream):
        self.original_stream = original_stream  # 保存传入的原始流引用
    
    def write(self, text):
        if text.strip():
            # 尝试解析uvicorn格式的日志
            if (
                text.startswith("INFO:")
                or text.startswith("WARNING:")
                or text.startswith("ERROR:")
            ):
                parts = text.strip().split(":", 1)
                if len(parts) == 2:
                    level = parts[0].strip()
                    message = parts[1].strip()
                    logger.bind(name="uvicorn", version=VERSION).info(message)
            else:
                # 区分是stderr还是stdout
                stream_type = "stderr" if self.original_stream == sys.__stderr__ else "stdout"
                logger.bind(name=stream_type, version=VERSION).warning(text.strip())
    
    def flush(self):
        self.original_stream.flush()
    
    def isatty(self):
        """实现isatty方法，委托给原始流"""
        try:
            return self.original_stream.isatty()
        except AttributeError:
            return False
```            

停机的时候印这个信息
```
251028 15:51:48[0.0.4][app.core.logger]-INFO-INFO:     Waiting for application shutdown.

251028 15:51:48[0.0.4][app.core.logger]-INFO-INFO:     Application shutdown complete.

251028 15:51:48[0.0.4][app.core.logger]-INFO-INFO:     Finished server process [66692]
```