---
title: "sigv4签名算法"
date: "2025-11-17"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

参考aws签名算法https://docs.aws.amazon.com/zh_cn/general/latest/gr/sigv4-calculate-signature.html  
请求头：  
Accept-AccessKey：PK  
Accept-Time：时间戳  
Accept-Nonce：混淆  
1.Accept-AccessKey|Accept-Time|uri|Accept-Nonce 拼接的字符串  
signKey 对 secretKey 进行一次HmacSHA256加密，拿到字符串kDate  
2.对body或param进行一次sha256加密，得到messags  
3.用 kDate 对 message 进行第二次HmacSHA256加密，拿到最终的Accept-Sign  
签名算法示例1（ak：AAAAAAAA-aaa sk:BBBBBBBBBBBBBBBBBBBB）：  
signKey: AAAAAAAA-aaa|1578982180608|/app/test|123 对secretKey第一次加密，得到kDate: 9fc748bffba8cff890c21197e7c6a0e00f9b751e723fd2a528bf10a44a096199  
body：{"data":{"key":"value"}}对body进行sha256加密得到message：6d03299fbba4e28d5b65007acecd643423db710bfdde6c6750d8dbf28f80fc8c用 kDate 对 message 进行第二次加密，得到最后的Accept-Sign：b1150c711526c26987a69236c11977cf0bb1817eff9ad1be4a286cf015a14a2d  

param：a=3&b=2&c=1 (字典序)  
对param进行sha256加密得到message：6ddd753a67129491c7183eb6b8ff2e4d8e7c5a35d31f7a491cd72455bcfe3244  
用 kDate 对 message 进行第二次加密，得到最后的Accept-Sign：c257939c96a398e778089ecf0394f0271d3f5446e38e7bf9ba15791b821272bb  

```
import org.apache.commons.codec.digest.DigestUtils;
import org.apache.commons.codec.digest.HmacAlgorithms;
import org.apache.commons.codec.digest.HmacUtils;

private static String generateSign(String ak, String requestTime,
                                   String uri, String nonce, String sk, String params) {
    StringBuffer stringBuffer = new StringBuffer(1024);
    stringBuffer.append(ak);
    stringBuffer.append("|");
    stringBuffer.append(requestTime);
    stringBuffer.append("|");
    stringBuffer.append(uri);
    stringBuffer.append("|");
    stringBuffer.append(nonce);
    //第一步根据时间戳做一次加密
    HmacUtils hmac = new HmacUtils(HmacAlgorithms.HMAC_SHA_256, stringBuffer.toString());
    byte[] kDate = hmac.hmac(sk);
    //请求参数进行sha256加密
    byte[] messageType = DigestUtils.sha256(params);

    //加密后的sk再对请求参数做一层加密
    hmac = new HmacUtils(HmacAlgorithms.HMAC_SHA_256, kDate);
    return hmac.hmacHex(messageType);
}
```