---
title: "python协程 异步编程"
date: "2025-10-31"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

- 协程，简而言之，是一种可以挂起和恢复执行的函数。与传统线程不同，协程是在同一线程内协作执行，无需切换上下文，因此开销极小。这使得协程非常适合处理大量并发的I/O密集型任务，如网络请求、文件读写等。协程是异步任务的基本单元，是一种用户态的上下文切换技术，其实就是通过一个线程实现代码块相互切换执行，本质是可暂停 / 恢复的函数，通过 async def 定义。与普通函数的区别在于：
调用协程不会立即执行，而是返回一个协程对象
- 在函数定义前加上 async 关键字时，这个函数就变成了一个"协程"（coroutine）
- 协程必须在异步上下文（如 async def 函数中通过 await 调用）才能执行 await 关键字等待一个协程完成
- 若 await 的是异步 I/O 操作（如网络请求、文件读取） 异步 I/O 操作（如 aiohttp 的请求、asyncio.open() 读取文件）通常由操作系统的 I/O 多路复用机制（如 select、epoll、kqueue）处理，无需占用 Python 线程。
- 若 await 的是其他协程（async def 函数） 被 await 的协程会被事件循环调度执行，此时执行权转移到该协程，直到它内部也遇到 await 并让出执行权。
- 若 await 的是同步函数（通过 run_in_threadpool 包装） 如果在异步方法中调用同步函数（如 CPU 密集型任务），通常会用 await run_in_threadpool(sync_func) 包装，此时同步函数会在线程池的 worker 线程中执行。
- await 在这里的作用是：事件循环等待线程池中的任务完成，期间会调度其他异步任务，不会阻塞事件循环

```
def run(self, sockets: list[socket.socket] | None = None) -> None:
    self.config.setup_event_loop()
    return asyncio.run(self.serve(sockets=sockets))

async def serve(self, sockets: list[socket.socket] | None = None) -> None:
    with self.capture_signals():
        await self._serve(sockets)

async def _serve(self, sockets: list[socket.socket] | None = None) -> None:
    process_id = os.getpid()

    config = self.config
    if not config.loaded:
        config.load()

    self.lifespan = config.lifespan_class(config)

    message = "Started server process [%d]"
    color_message = "Started server process [" + click.style("%d", fg="cyan") + "]"
    logger.info(message, process_id, extra={"color_message": color_message})

    await self.startup(sockets=sockets)
    if self.should_exit:
        return
    await self.main_loop()
    await self.shutdown(sockets=sockets)

    message = "Finished server process [%d]"
    color_message = "Finished server process [" + click.style("%d", fg="cyan") + "]"
    logger.info(message, process_id, extra={"color_message": color_message})
```

asyncio.run(self.serve(sockets=sockets))  
self.serve(sockets=sockets) 就是返回一个携程  
asyncio.run() 创建事件循环，运行 返回的协程，直到它完成  
await self._serve(sockets) 就是等待完成才会走下去  
不能在运行的事件循环中调用 asyncio.run()  

run_in_executor 使用线程池调用一个同步方法  

run_in_threadpool 本质上是对  
asyncio.get_running_loop().run_in_executor() 的封装。  
async def + await   
await后面跟着定义成async def 的方法 不允许又同步操作 都得改成await使用异步方法  
异步接口的 TPS 不取决于 “单线程每秒能跑多少个完整请求”，而取决于 “每个请求的实际 CPU 占用时间” 和 “IO 等待时间的占比”：  
假设每个接口总耗时 0.1s，其中 IO 等待时间 0.09s（不占用 CPU），CPU 计算时间 0.01s（占用主线程）。  
主线程 1 秒内可处理的 “CPU 计算时间” 是 1s，因此理论 TPS = 1s / 0.01s = 100（忽略事件循环调度开销）。  
如果 IO 等待时间更长（如 0.095s，CPU 时间 0.005s），理论 TPS 可达到 200。  
这就是异步框架处理 IO 密集型场景（如 API 调用、数据库查询、文件读写）的核心优势 —— 用单线程 “并行利用” 所有请求的 IO 等待时间。  
如果你的接口是 CPU 密集型（如 0.1s 全是 CPU 计算，无任何 await 等待），此时异步模型确实会退化为 “单线程串行” 得改成用def 这时候整个方法都会扔到线程池里面去  
如果def方法里面又同步方法 无所谓 反正def方法已经是同步的了 但是你想def 接口中调用 run_in_threadpool 方法去做一个异步是做不到的 因为加不上await 他会阻塞当前子线程（等待 IO 完成），只是将阻塞操作从 “当前线程” 转移到了 “线程池的另一个线程”，本质上还是消耗线程资源，无法达到异步 IO 的效率。  
所以一旦同步之后后面都只能同步下去 除非一直异步下去 所以一般使用async def异步 前提是你是io密集型任务  
