---
title: "HTTP长轮询"
date: "2019-07-06"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
可以使用servlet 3.0 异步处理来实现长轮询。

```
// SendMessageServlet.java - 客户端发信息的入口
@WebServlet("/api/send")
public class SendMessageServlet extends HttpServlet {
    
    // POST请求：发送消息
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) 
            throws ServletException, IOException {
        
        // 1. 获取请求参数
        String from = req.getParameter("from");      // 发送者ID
        String to = req.getParameter("to");          // 接收者ID（可选）
        String content = req.getParameter("content"); // 消息内容
        String roomId = req.getParameter("roomId");   // 聊天室ID（可选）
        
        // 2. 参数验证
        if (from == null || from.trim().isEmpty() || 
            content == null || content.trim().isEmpty()) {
            sendError(resp, 400, "缺少必要参数");
            return;
        }
        
        // 3. 创建消息对象
        Message message = new Message(from, to, content);
        if (roomId != null) {
            message.setRoomId(roomId);
        }
        
        // 4. 发送消息
        MessageDispatcher dispatcher = MessageDispatcher.getInstance();
        MessageDispatcher.SendResult result = dispatcher.sendMessage(message);
        
        // 5. 返回结果
        resp.setContentType("application/json;charset=UTF-8");
        PrintWriter writer = resp.getWriter();
        
        JsonObject json = Json.createObjectBuilder()
            .add("status", "success")
            .add("messageId", message.getId())
            .add("delivered", result.getDeliveredCount())
            .add("timestamp", message.getTimestamp())
            .build();
        
        writer.write(json.toString());
    }
    
    private void sendError(HttpServletResponse resp, int code, String message) 
            throws IOException {
        resp.setStatus(code);
        resp.setContentType("application/json");
        resp.getWriter().write("{\"error\":\"" + message + "\"}");
    }
}
```
客户端拉取消息接口
```
// PollMessageServlet.java - 客户端接收消息（长轮询）
@WebServlet(value = "/api/poll", asyncSupported = true)
public class PollMessageServlet extends HttpServlet {
    
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) 
            throws ServletException, IOException {
        
        // 1. 获取客户端标识
        String clientId = req.getParameter("clientId");
        if (clientId == null || clientId.trim().isEmpty()) {
            resp.sendError(400, "缺少clientId参数");
            return;
        }
        
        // 2. 启动异步处理
        AsyncContext asyncContext = req.startAsync();
        
        // 3. 设置超时时间（长轮询超时）
        asyncContext.setTimeout(45000); // 45秒
        
        // 4. 注册监听器
        asyncContext.addListener(new AsyncListener() {
            @Override
            public void onComplete(AsyncEvent event) {
                System.out.println("客户端 " + clientId + " 连接完成");
            }
            
            @Override
            public void onTimeout(AsyncEvent event) {
                try {
                    // 超时返回空响应
                    HttpServletResponse response = 
                        (HttpServletResponse) event.getAsyncContext().getResponse();
                    response.setContentType("application/json");
                    response.getWriter().write("{\"status\":\"timeout\"}");
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    event.getAsyncContext().complete();
                    MessageDispatcher.getInstance().removeWaitingClient(clientId);
                }
            }
            
            @Override
            public void onError(AsyncEvent event) {
                MessageDispatcher.getInstance().removeWaitingClient(clientId);
                event.getAsyncContext().complete();
            }
            
            @Override
            public void onStartAsync(AsyncEvent event) {}
        });
        
        // 5. 将客户端加入等待队列
        MessageDispatcher.getInstance().addWaitingClient(clientId, asyncContext);
    }
}
```
