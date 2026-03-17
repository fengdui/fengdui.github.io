---
title: "silicon文本转图片"
date: "2026-03-16"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

```
async def send_message_proxy(app: Ariadne, group: Union[Group, Friend], resp: MessageChain, quote: Union[bool, MessageChain]=False):
    msg = await app.send_message(group,resp,quote=quote)
    if msg.source.id < 0:
        txt = '\n'.join([textwrap.fill(i) for i in resp.display.splitlines()])
        with open('in.txt','wb') as f:
            f.write(txt.encode('utf-8'))
        os.system('silicon.exe in.txt -o out.png -f "微软雅黑" -l c --no-window-controls --background "#fff0" --pad-horiz 0 --pad-vert 0 --no-line-number --no-round-corner 2>nul')
        with open('out.png','rb') as f:
            pic = f.read()
        await app.send_message(group,MessageChain(Image(data_bytes=pic)),quote=quote)
    return
```
https://carbon.now.sh  