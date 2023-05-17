# 一、RSA简介

## 1、加密解密

RSA加密是一种非对称加密，在公开密钥加密和电子商业中RSA被广泛使用。可以在不直接传递密钥的情况下，完成加解密操作。这能够确保信息的安全性，避免了直接传递密钥所造成的被泄露的风险。是由一对密钥来进行加解密的过程，分别称为公钥和私钥。该加密算法的原理就是对一极大整数做因数分解的困难性来保证安全性。

## 2、签名验签

数字签名就是信息的来源添加一段无法被伪造的加密字符串，这段数字串作为对信息的来源真实性的一个有效证明。这个过程称为签名和验签。

# 二、场景描述

- 消息发送方：甲方，持有公钥
- 消息接收方：乙方，持有私钥

## 1、加密解密过程

(1)、乙方生成一对密钥即公钥和私钥，私钥不公开，乙方自己持有，公钥为公开，甲方持有。

(2)、乙方收到甲方加密的消息，使用私钥对消息进行解密，获取明文。

## 2、签名验签过程

(1)、乙方收到消息后，需要回复甲方，用私钥对回复消息签名，并将消息明文和消息签名回复甲方。

(2)、甲方收到消息后，使用公钥进行验签，如果验签结果是正确的，则证明消息是乙方回复的。


# 三、源代码实现

## 1、密钥字符串获取

- 代码生成
```java
private static HashMap<String, String> getTheKeys() {
    HashMap<String, String> keyPairMap = new HashMap<String, String>();
    KeyPairGenerator keyPairGen = null;
    try {
        keyPairGen = KeyPairGenerator.getInstance("RSA");
    } catch (NoSuchAlgorithmException e) {
        e.printStackTrace();
    }
    // 密钥大小：1024 位
    keyPairGen.initialize(1024);
    KeyPair keyPair = keyPairGen.generateKeyPair();
    String publicKey = printBase64Binary(keyPair.getPublic().getEncoded());
    String privateKey = printBase64Binary(keyPair.getPrivate().getEncoded());
    keyPairMap.put("publicKey", publicKey);
    keyPairMap.put("privateKey", privateKey);
    return keyPairMap ;
}
```
- 读取文件

![](https://images.gitee.com/uploads/images/2022/0221/224256_27d2641c_5064118.png "01-1.png")

文件位置
```java
public static final String PUB_KEY = "rsaKey/public.key" ;
public static final String PRI_KEY = "rsaKey/private.key" ;
```

文件加载
```java
public static String getKey (String keyPlace) throws Exception {
    BufferedReader br= null;
    try {
        br= new BufferedReader(new InputStreamReader(RsaCryptUtil.class.getClassLoader().
                                                     getResourceAsStream(keyPlace)));
        String readLine= null;
        StringBuilder keyValue = new StringBuilder();
        while((readLine= br.readLine())!=null){
            if(!(readLine.charAt(0)=='-')){
                keyValue.append(readLine);
            }
        }
        return keyValue.toString();
    } catch (Exception e) {
        throw new Exception("RSA密钥读取错误",e) ;
    } finally{
        if (br != null) {
            try {
                br.close();
            } catch (Exception e) {
                System.out.println("密钥读取流关闭异常");
            }
        }
    }
}
```

## 2、公钥和私钥

- 公钥字符串生成公钥
```java
public static RSAPublicKey createPublicKey(String publicKeyValue) throws Exception {
    try {
        byte[] buffer = DatatypeConverter.parseBase64Binary(publicKeyValue);
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
        X509EncodedKeySpec keySpec = new X509EncodedKeySpec(buffer);
        return (RSAPublicKey) keyFactory.generatePublic(keySpec);
    } catch (Exception e) {
        throw new Exception("公钥创建失败", e);
    }
}
```

- 私钥字符串生成私钥
```java
public static RSAPrivateKey createPrivateKey(String privateKeyValue) throws Exception {
    try {
        byte[] buffer = javax.xml.bind.DatatypeConverter.parseBase64Binary(privateKeyValue);
        PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(buffer);
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
        return (RSAPrivateKey) keyFactory.generatePrivate(keySpec);
    } catch (Exception e) {
        throw new Exception("私钥创建失败", e);
    }
}
```

## 3、加密和解密

- 公钥加密
```java
public static String encrypt(RSAPublicKey publicKey, byte[] clearData) throws Exception {
    if (publicKey == null) {
        throw new Exception("加密公钥为空, 无法加密");
    }
    try {
        Cipher cipher = Cipher.getInstance("RSA") ;
        cipher.init(Cipher.ENCRYPT_MODE, publicKey);
        byte[] output = cipher.doFinal(clearData);
        return printBase64Binary(output);
    } catch (Exception e) {
        throw new Exception("公钥加密失败",e);
    }
}
```

- 私钥解密
```java
public static String decrypt(RSAPrivateKey privateKey, byte[] cipherData) throws Exception {
    if (privateKey == null) {
        throw new Exception("解密私钥为空, 无法解密");
    }
    try {
        Cipher cipher = Cipher.getInstance("RSA") ;
        cipher.init(Cipher.DECRYPT_MODE, privateKey);
        byte[] output = cipher.doFinal(cipherData);
        return new String(output);
    } catch (BadPaddingException e) {
        throw new Exception("私钥解密失败",e);
    }
}
```

## 4、签名和验签

- 私钥签名
```java
public static String sign (String signData, PrivateKey privateKey) throws Exception {
    byte[] keyBytes = privateKey.getEncoded();
    PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(keyBytes);
    KeyFactory keyFactory = KeyFactory.getInstance("RSA");
    PrivateKey key = keyFactory.generatePrivate(keySpec);
    Signature signature = Signature.getInstance("MD5withRSA");
    signature.initSign(key);
    signature.update(signData.getBytes());
    return printBase64Binary(signature.sign());
}
```

- 公钥验签
```java
public static boolean verify(String srcData, PublicKey publicKey, String sign) throws Exception {
    byte[] keyBytes = publicKey.getEncoded();
    X509EncodedKeySpec keySpec = new X509EncodedKeySpec(keyBytes);
    KeyFactory keyFactory = KeyFactory.getInstance("RSA");
    PublicKey key = keyFactory.generatePublic(keySpec);
    Signature signature = Signature.getInstance("MD5withRSA");
    signature.initVerify(key);
    signature.update(srcData.getBytes());
    return signature.verify(parseBase64Binary(sign));
}
```

## 5、编码和解码
```java
/**
 * 字节数组转字符
 */
public static String printBase64Binary(byte[] bytes) {
    return DatatypeConverter.printBase64Binary(bytes);
}
/**
 * 字符转字节数组
 */
public static byte[] parseBase64Binary(String value) {
    return DatatypeConverter.parseBase64Binary(value);
}
```

## 6、测试代码块

- 密钥生成测试
```java
public static void testCreateKey () throws Exception {
    HashMap<String, String> map = RsaCryptUtil.getTheKeys();
    String privateKeyStr=map.get("privateKey");
    String publicKeyStr=map.get("publicKey");
    System.out.println("私钥："+privateKeyStr);
    System.out.println("公钥："+publicKeyStr);
    //消息发送方
    String originData="cicada-smile";
    System.out.println("原文："+originData);
    String encryptData = RsaCryptUtil.encrypt(RsaCryptUtil.createPublicKey(publicKeyStr),
                                              originData.getBytes());
    System.out.println("加密："+encryptData);
    //消息接收方
    String decryptData=RsaCryptUtil.decrypt(RsaCryptUtil.createPrivateKey(privateKeyStr),
                                            RsaCryptUtil.parseBase64Binary(encryptData));
    System.out.println("解密："+decryptData);
}
```

- 密钥读取测试
```java
public static void testReadKey () throws Exception {
    String value = getKey("rsaKey/public.key");
    System.out.println(value);
    String privateKeyStr = getKey(RsaCryptUtil.PRI_KEY) ;
    String publicKeyStr = getKey(RsaCryptUtil.PUB_KEY) ;
    //消息发送方
    String originData="cicada-smile";
    System.out.println("原文："+originData);
    String encryptData = RsaCryptUtil.encrypt(RsaCryptUtil.createPublicKey(publicKeyStr),
                                              originData.getBytes());
    System.out.println("加密："+encryptData);
    //消息接收方
    String decryptData=RsaCryptUtil.decrypt(RsaCryptUtil.createPrivateKey(privateKeyStr),
                                            RsaCryptUtil.parseBase64Binary(encryptData));
    System.out.println("解密："+decryptData);
}
```

- 签名验签测试
```java
public static void testSignVerify () throws Exception {
    String signData = "cicada-smile" ;
    String privateKeyStr = getKey(RsaCryptUtil.PRI_KEY) ;
    String publicKeyStr = getKey(RsaCryptUtil.PUB_KEY) ;
    String signValue = sign(signData,RsaCryptUtil.createPrivateKey(privateKeyStr)) ;
    boolean flag = verify(signData,RsaCryptUtil.createPublicKey(publicKeyStr),signValue);
    System.out.println("原文:"+signData);
    System.out.println("签名:"+signValue);
    System.out.println("验签:"+flag);
}
```

**源码参考：** https://gitee.com/cicadasmile/model-arithmetic-parent