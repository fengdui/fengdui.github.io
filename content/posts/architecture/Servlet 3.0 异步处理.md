---
title: "Servlet 3.0 异步处理"
date: "2015-09-26"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
传统同步流程：

客户端请求 → 分配线程 → 执行doGet/doPost → 返回响应 → 释放线程
- 200线程 → 最多同时处理200请求
- 大部分线程在等待IO，CPU闲置
- 线程上下文切换开销大
- 无法应对高并发（如10000+连接）

异步流程：
客户端请求 → 分配线程 → 启动异步 → 立即释放线程 →
后台处理 → 完成时重新获取线程 → 返回响应 → 释放线程

关键区别：启动异步后，容器线程立即释放

```
// 传统同步 Servlet
@WebServlet("/sync")
public class SyncServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) 
            throws IOException {
        
        // 线程被阻塞在这里！
        String data = callExternalService(); // 耗时2秒
        String dbResult = queryDatabase();   // 耗时1秒
        
        resp.getWriter().write(data + " - " + dbResult);
        // 线程总共被占用3秒
    }
}

// 异步 Servlet
@WebServlet(value = "/async", asyncSupported = true)
public class AsyncServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
        AsyncContext asyncContext = req.startAsync();
        
        // 提交到线程池，立即释放容器线程
        CompletableFuture.runAsync(() -> {
            try {
                String data = callExternalService(); // 2秒
                String dbResult = queryDatabase();   // 1秒
                
                ServletResponse response = asyncContext.getResponse();
                response.getWriter().write(data + " - " + dbResult);
                
            } catch (Exception e) {
                // 错误处理
            } finally {
                asyncContext.complete();
            }
        });
        
        // 方法立即返回，容器线程只用了约1毫秒！
    }
}
```
物理层面：NIO 使用多路复用，一个选择器线程可以管理成千上万个连接

应用层面：每个客户端连接在业务逻辑上对应一个 AsyncContext 对象

映射关系：Selector → (多个 SelectionKey) → (多个 SocketChannel) → (多个 AsyncContext)

核心价值：AsyncContext 解耦了线程和连接，使得应用可以用少量线程服务大量并发连接