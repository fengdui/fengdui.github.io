---
title: "gpg加密"
date: "2020-05-27"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

用户将自己的公钥上传  
当用户下载时使用公钥加密 用户使用自己的私钥解密

加密只需要公钥
解密的时候需要私钥和对应的密码

```
package com.hodgepodge.framework.encrypt;

import lombok.extern.slf4j.Slf4j;
import org.bouncycastle.bcpg.ArmoredInputStream;
import org.bouncycastle.jce.provider.BouncyCastleProvider;
import org.bouncycastle.openpgp.*;
import org.bouncycastle.openpgp.jcajce.JcaPGPObjectFactory;
import org.bouncycastle.openpgp.operator.jcajce.*;
import org.bouncycastle.util.io.Streams;
import org.springframework.util.StringUtils;

import java.io.*;
import java.security.NoSuchProviderException;
import java.security.SecureRandom;
import java.security.Security;
import java.util.Date;
import java.util.Iterator;

@Slf4j
public class GPGUtil {

    static {
        Security.addProvider(new BouncyCastleProvider());
    }

    /**
     * 预处理数据 转成二进制格式等
     *
     * @param bytes
     * @param fileName
     * @return
     * @throws IOException
     */
    private static byte[] preProcessData(byte[] bytes, String fileName) {
        PGPLiteralDataGenerator lData = new PGPLiteralDataGenerator();
        try (ByteArrayOutputStream bOut = new ByteArrayOutputStream();
             OutputStream pOut = lData.open(bOut, PGPLiteralData.BINARY, fileName, bytes.length, new Date())) {
            pOut.write(bytes);
            return bOut.toByteArray();
        } catch (Exception e) {
            log.error("GPG加密失败", e);
            throw new RuntimeException("内部错误");
        }
    }

    /**
     * 加密一个字节数组
     *
     * @param
     * @return
     */
    public static byte[] encryptData(byte[] bytes, String fileName, String pubKeyStr) throws IOException, PGPException {

        if (StringUtils.isEmpty(pubKeyStr)) {
            throw new RuntimeException("pubKeyStr is empty");
        }
        bytes = preProcessData(bytes, fileName);
        PGPPublicKey encKey = readPublicKey(pubKeyStr);
        PGPEncryptedDataGenerator encGen = new PGPEncryptedDataGenerator(
                new JcePGPDataEncryptorBuilder(PGPEncryptedData.CAST5)
                        .setWithIntegrityPacket(true)
                        .setSecureRandom(new SecureRandom()));

        encGen.addMethod(new JcePublicKeyKeyEncryptionMethodGenerator(encKey));

        try (ByteArrayOutputStream out = new ByteArrayOutputStream();
             OutputStream cOut = encGen.open(out, bytes.length)) {
            cOut.write(bytes);
            cOut.close();
            out.close();
            return out.toByteArray();
        } catch (Exception e) {
            log.error("GPG加密失败", e);
            throw new RuntimeException("系统内部错误");
        }
    }

    /**
     * 将一个公钥转成PGPPublicKey
     *
     * @param pubKeyStr
     * @return
     * @throws IOException
     * @throws PGPException
     */
    public static PGPPublicKey readPublicKey(String pubKeyStr) throws IOException, PGPException {
        if (StringUtils.isEmpty(pubKeyStr)) {
            throw new IllegalArgumentException("pubKey is empty");
        }
        try (ArmoredInputStream keyIn = new ArmoredInputStream(new ByteArrayInputStream(pubKeyStr.getBytes("UTF-8")))) {
            PGPPublicKey pubKey = readPublicKey(keyIn);
            return pubKey;
        }
    }

    /**
     * 从一个文件中PGPPublicKey
     *
     * @param fileName
     * @return
     * @throws IOException
     * @throws PGPException
     */
    public static PGPPublicKey readPublicKeyFormFile(String fileName) throws IOException, PGPException {
        if (StringUtils.isEmpty(fileName)) {
            throw new IllegalArgumentException("fileName is empty");
        }
        try (BufferedInputStream keyIn = new BufferedInputStream(new FileInputStream(fileName))) {
            PGPPublicKey pubKey = readPublicKey(keyIn);
            log.info("find PGPPublicKey, fileName={}", fileName);
            return pubKey;
        }
    }

    public static PGPPublicKey readPublicKey(InputStream input) throws IOException, PGPException {
        PGPPublicKeyRingCollection pgpPub = new PGPPublicKeyRingCollection(
                PGPUtil.getDecoderStream(input), new JcaKeyFingerprintCalculator());

        Iterator<PGPPublicKeyRing> keyRingIter = pgpPub.getKeyRings();
        while (keyRingIter.hasNext()) {
            PGPPublicKeyRing keyRing = keyRingIter.next();

            Iterator keyIter = keyRing.getPublicKeys();
            while (keyIter.hasNext()) {
                PGPPublicKey key = (PGPPublicKey) keyIter.next();

                if (key.isEncryptionKey()) {
                    return key;
                }
            }
        }

        throw new IllegalArgumentException("Can't find encryption key in key ring.");
    }

    /**
     * 解密一个文件
     *
     * @param inputFileName
     * @param keyFileName
     * @param passwd
     * @param defaultFileName
     * @throws IOException
     * @throws NoSuchProviderException
     */
    private static void decryptData(String inputFileName, String keyFileName, char[] passwd, String defaultFileName) throws IOException {
        InputStream in = new BufferedInputStream(new FileInputStream(inputFileName));
        InputStream keyIn = new BufferedInputStream(new FileInputStream(keyFileName));
        decryptData(in, keyIn, passwd, defaultFileName);
        keyIn.close();
        in.close();
    }


    private static PGPPrivateKey findSecretKey(PGPSecretKeyRingCollection pgpSec, long keyID, char[] pass)
            throws PGPException {
        PGPSecretKey pgpSecKey = pgpSec.getSecretKey(keyID);

        if (pgpSecKey == null) {
            return null;
        }

        return pgpSecKey.extractPrivateKey(new JcePBESecretKeyDecryptorBuilder().setProvider("BC").build(pass));
    }

    public static void decryptData(InputStream in, InputStream keyIn, char[] passwd, String defaultFileName) throws IOException {
        in = PGPUtil.getDecoderStream(in);

        try {
            JcaPGPObjectFactory pgpF = new JcaPGPObjectFactory(in);
            PGPEncryptedDataList enc;

            Object o = pgpF.nextObject();
            if (o instanceof PGPEncryptedDataList) {
                enc = (PGPEncryptedDataList) o;
            } else {
                enc = (PGPEncryptedDataList) pgpF.nextObject();
            }
            Iterator it = enc.getEncryptedDataObjects();
            PGPPrivateKey sKey = null;
            PGPPublicKeyEncryptedData pbe = null;
            PGPSecretKeyRingCollection pgpSec = new PGPSecretKeyRingCollection(
                    PGPUtil.getDecoderStream(keyIn), new JcaKeyFingerprintCalculator());

            while (sKey == null && it.hasNext()) {
                pbe = (PGPPublicKeyEncryptedData) it.next();

                sKey = findSecretKey(pgpSec, pbe.getKeyID(), passwd);
            }

            if (sKey == null) {
                throw new IllegalArgumentException("secret key for message not found.");
            }

            InputStream clear = pbe.getDataStream(new JcePublicKeyDataDecryptorFactoryBuilder().setProvider("BC").build(sKey));

            JcaPGPObjectFactory plainFact = new JcaPGPObjectFactory(clear);

            Object message = plainFact.nextObject();

            if (message instanceof PGPCompressedData) {
                PGPCompressedData cData = (PGPCompressedData) message;
                JcaPGPObjectFactory pgpFact = new JcaPGPObjectFactory(cData.getDataStream());

                message = pgpFact.nextObject();
            }

            if (message instanceof PGPLiteralData) {
                PGPLiteralData ld = (PGPLiteralData) message;

                String outFileName = ld.getFileName();
                if (outFileName.length() == 0) {
                    outFileName = defaultFileName;
                }

                InputStream unc = ld.getInputStream();
                OutputStream fOut = new BufferedOutputStream(new FileOutputStream(outFileName));

                Streams.pipeAll(unc, fOut);

                fOut.close();
            } else if (message instanceof PGPOnePassSignatureList) {
                throw new PGPException("encrypted message contains a signed message - not literal data.");
            } else {
                throw new PGPException("message is not a simple encrypted file - type unknown.");
            }
        } catch (PGPException e) {
            log.error("解密失败", e);
            if (e.getUnderlyingException() != null) {
                e.getUnderlyingException().printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws IOException, PGPException {
        Security.addProvider(new BouncyCastleProvider());
        byte[] bytes = encryptData("1234567890qwwwwweee\n".getBytes("UTF-8"), "/Users/fd/fd-en.txt", "");
        File file = new File("/Users/fd/fd-en");
        file.createNewFile(); // 创建新文件
        BufferedOutputStream out = new BufferedOutputStream(new FileOutputStream(file));
        out.write(bytes);
        out.close();


        decryptData("/Users/fd/fd-en", "/Users/fd/.gnupg/secring.gpg", "123456".toCharArray(), "/Users/fd/rrrr");
    }
}
```