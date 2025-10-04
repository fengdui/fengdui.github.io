---
title: "libpcap+bpf抓包进行数据库发现"
date: "2024-04-08"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

大致流程：
数据库流量镜像到网卡，c++程序对到网卡的流量进行抓包分析，协议解析以确定是哪一种数据库。

遍历网卡

```js
if (pcap_findalldevs(&alldevs, errbuf) == -1) {
        return false;
    }
```

打开网络接口

```js
if ((pd_ = pcap_open_live(d->name, // name of the device
                              65536,   // portion of the packet to capture.
                                       // 65536 grants that the whole packet will be captured on all the MACs.
                              1,       // promiscuous mode (nonzero means promiscuous)
                              1000,    // read timeout
                              errbuf   // error buffer
                              ))
```
关闭

```js
        pcap_close(t_pd);
```
free释放空间

```js
    pcap_freealldevs(alldevs);

```
获取数据包

```js
    int ret = pcap_loop(pd_, 0, onPacket, (unsigned char*) this);
```
onPacket是回调函数 回调之后解包分析


编译表达式和设置过滤器 例："port 443"，"src host 127.0.0.1";

```js
if (pcap_compile(pd_, &prog, filter.c_str(), 1, 0) < 0) {
        last_error_ = util::format("pcap_compile, %s : %s", pcap_geterr(pd_), filter.c_str());
        return false;
    }
```

mysql协议解析

```js
int32_t MySQLProtocol::dissector(const uint8_t* buf, int32_t size) {
    if (size <= 4) {
        return RET_UNKNOWN;
    }

    int32_t packet_len = unmarshall_number<int32_t, 3>((char*) buf, false) + 4;
    if (packet_len != size) {
        return RET_UNKNOWN;
    }

    uint8_t packet_number = (uint8_t) buf[3];
    uint8_t server_version = (uint8_t) buf[5];

    if (phase_ == PHASE_FULL_LINK && packet_number == 1) {
        return RET_C2S;
    }

    uint8_t flag = (uint8_t) buf[4];
    if (flag == 0 || flag == 0xFE || flag == 0xFF) {
        return RET_S2C;
    }
    if (flag == 1 && packet_len == 1) {
        return RET_C2S;
    }
    if (flag > 1 && flag < 0x1f) {
        set_proto_id_by_default();
        return RET_C2S;
    }

    return RET_UNKNOWN;
}
```


https://blog.csdn.net/weiyong1999/article/details/8657455  
https://biot.com/capstats/bpf.html