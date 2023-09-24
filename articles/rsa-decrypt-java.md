---
title: "JavaでRSA暗号の復号化を細かく辿っていく"
emoji: ""
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["java", "rsa"]
published: true
published_at: 2024-09-24 01:00
---

# はじめに
以前「RSA暗号化してデータを送信してるんだけど、送信先が不正データとして扱うので原因調査して」という依頼がありました。
「そんなの分かるわけないだろ…」と思いながら試行錯誤していくと結果として解決できましたので、その思考/試行の流れを記載します。

# 調査方針
## 本題の前に
調査方針に入る前に問題です。
以下2つのテキストのうち、一方が正常なRSA暗号化を施したテキスト、もう片方が不正データとなるRSA暗号化を施したテキストです。
どちらが正常なRSA暗号化を施したテキストでしょうか。

- `cFKJQ9X7UYdFOvHkERKE3E6Y2o8HU8bHAihFL0vsb2cYnyd8aQ/2zCels0ZcxWS/nAamfFS2KfmTmVels0mQFsA0QK4vS6EGAfkLwsUyXW9w/8CJp9byGYFHTKbgML6/s7o8Cq5p5E9VdQW++B5CZKD0Bedaz4p4htEw2sL2yXhO4k+NGPAQvbwwRaw8NEAs/w1E9/+hhrTIGognkCFHtSn8VHup3xaPmyg2bldb0GmH8Gceof7PMX7RJKP5oc4LZ3v44DesOGHc8dUqji+8OP/ogpu1BZFFbzL+9arQtM2hnbGeB5188wmLrq4vGSs99ofdHGjvdrkrJWhma2e/bQ==`
- `XaayapgtFe9K2HUd39XPdc0F5PWCIMEM/XWZAFxlGEPcsHMNwNFg5qJS0l1bbM6qe2tpVplHFWgjxxGFyLZQ/In41cDAOKqvMZ1juGw7KlR1Pso3UvbpnZmACZ0kuaS/HYj+h4XrNQZg4fj9TsspLHd4fhPGk+B84aJuPPG24uGHastIH/6BgXxAE98mlNKJ4TOu7T49kebbgH6MgSsIUQYOP69y+F2lfCel4Pq4EWVc4Dqc9ck7faBySMfzmPkLkz4pfkXatgD8sU17mjD/URP1msnkYDBkYFoKuyhuHX6zBz5yhLDSSTWL8tmCOody3/y9mbQkX/CO9c+gz/EBdQ==`
 
ちなみに正解は下です。
………分かるわけない？全く同感です。

つまり「RSA暗号化して出てきたテキストがこれなんだけど、送信しても不正データ扱いされるの」と言って上のテキストを渡されても、「何も分からん」と返すしかないのです。
もちろんRSA暗号用の公開鍵や元となるテキストを付けられても一緒です。暗号化してどのような形になったか分からないと本題の「なぜ不正データ扱いされるか」が分かりませんからね。

ということで、問題解決するには「どのような暗号化をされたか」をつかむ必要があります。

## RSA暗号化の内容
今回のRSA暗号化はPKCS#1 v1.5パディングという形式で行われていました。
詳細は[RFC 8017](https://tex2e.github.io/rfc-translater/html/rfc8017.html)を見てもらえればと思いますが、「7.2.1. 暗号化操作」節で暗号化されるメッセージのバイト列は以下のようになると記載されています。

```
EM = 0x00 || 0x02 || PS || 0x00 || M
```

先頭から順に

- 0x00, 0x02
- PS(ランダムなバイト列)
- 0x00
- メッセージ本体

で成り立っています。
つまり、RSA復号化を行った後のバイト列が上記のようになれば正常、そうでなければ異常となるはずです。

## JavaでのRSA復号化
JavaでRSA（PKCS#1 v1.5パディング）を復号するのは以下のコードでできます。（内容抜粋）

```java
String allPrivateKey = "..."; //ここに秘密鍵を記載
String encryptString = "..."; //ここに暗号化されたテキストを記載
byte[] encryptbuffer = Base64.decodeBase64(encryptString.getBytes());

byte[] buffer = Base64.decodeBase64(allPrivateKey.getBytes());
PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(buffer);
KeyFactory keyFactory = KeyFactory.getInstance("RSA");
PrivateKey privateKey = keyFactory.generatePrivate(keySpec);

Cipher cipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
cipher.init(Cipher.DECRYPT_MODE, privateKey);
String decryptedString = new String(cipher.doFinal(encryptbuffer));
```

しかし不正データとされるテキストをこのコードに渡しても、`javax.crypto.BadPaddingException: Decryption error`としか返さないためパディングの部分がNG判定されていることしか分かりません。
`e.printStackTrace();`でスタックトレースを出力させるとこうなりました。

```
javax.crypto.BadPaddingException: Decryption error
    at java.base/sun.security.rsa.RSAPadding.unpadV15(RSAPadding.java:369)
    at java.base/sun.security.rsa.RSAPadding.unpad(RSAPadding.java:282)
    at java.base/com.sun.crypto.provider.RSACipher.doFinal(RSACipher.java:371)
    at java.base/com.sun.crypto.provider.RSACipher.engineDoFinal(RSACipher.java:405)
    at java.base/javax.crypto.Cipher.doFinal(Cipher.java:2202)
    at com.test.App.decryptInfo(App.java:63)
    at com.test.App.main(App.java:29)
```

処理としては「①暗号化テキストをRSA復号化→②パディング除去→③復号化後のメッセージ部分を出力」としているのは間違いないだろうと目星をつけ、「スタックトレースに表示されるモジュールの中身を見ていけば暗号化テキストをRSA復号化した直後のバイト列を取得するコードを作成できるのではないか。バイト列さえ取得できればそれがパディングのルールに従っているか否か見ればよい」と判断しました。

# 実際のコード
こうして書き上げたコードが以下になります。

```java:App.java
import java.security.KeyFactory;
import java.security.PrivateKey;
import java.security.spec.PKCS8EncodedKeySpec;
import javax.crypto.Cipher;
import org.apache.commons.codec.binary.Base64;
import java.lang.String;
import java.security.interfaces.RSAPrivateKey;
import sun.security.rsa.*;

public class App 
{
    private static final String PG_PRIVATE_KEY = "..."; //ここに秘密鍵を記載
    public static void main(String[] args){
        try {
            // RSA復号化を実施
            System.out.println(decryptInfo());
        } catch (Exception e) {
            // 復号エラーとなった場合はスタックトレース表示
            e.printStackTrace();
        }
    }

    public static String decryptInfo() throws Exception{
        String encryptString = "..."; //RSA暗号化テキストを記載
        byte[] encryptbuffer = Base64.decodeBase64(encryptString.getBytes());

        byte[] buffer = Base64.decodeBase64(PG_PRIVATE_KEY.getBytes());
        PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(buffer);
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
        PrivateKey privateKey = keyFactory.generatePrivate(keySpec);
        RSAPrivateKey rsaprivatekey = (RSAPrivateKey)privateKey;
        
        // 復号後、パディング除去前のバイト列確認
        byte[] decryptBuffer = RSACore.convert(encryptbuffer, 0, encryptbuffer.length);
        byte[] data = RSACore.rsa(decryptBuffer, rsaprivatekey, false);
        System.out.print("bytes: ");
        // 復号後、パディング除去前のバイト列出力
        for(byte b: data){
            System.out.printf("%x ", b);
        }
        System.out.println();
        // 復号後、パディング除去前の文字列出力
        String converted = new String(data);
        System.out.println(converted);

        // 実際に復号後のメッセージを出力させ、復号、パディング除去可否の検証
        Cipher cipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
        cipher.init(Cipher.DECRYPT_MODE, privateKey);
        String decryptedString = new String(cipher.doFinal(encryptbuffer));

        return decryptedString;
    }
}
```

## 実行結果
「本題の前に」節に記載した2つのRSA暗号化後テキストを入力にするとこうなります。

- 上のテキスト（不正データ扱い）
```
bytes: 0 0 2 33 2d 75 7b 6c 48 63 74 77 2a 6c 64 54 41 2f 4f 65 51 7c 67 59 43 35 4f 7c 74 54 63 5a 32 3f 33 37 7d 77 27 36 43 40 34 70 4d 75 77 7b 31 32 73 40 7c 66 7e 6b 7a 7c 3c 69 70 48 3c 62 41 38 47 3e 69 6c 26 53 7c 3a 6c 53 23 4f 63 78 75 2b 4d 4c 7a 62 4c 2e 4a 77 65 2a 24 32 49 3d 7f 7c 69 39 73 47 45 23 65 2a 6a 60 4a 34 61 58 64 5f 59 6c 6c 58 7b 3c 53 26 2d 73 35 66 4f 46 53 6e 38 7d 77 2a 4d 76 3f 39 68 40 47 39 52 26 22 5b 5e 7c 70 68 35 4a 58 21 6e 4a 75 41 43 68 24 56 6f 4d 2e 3a 50 21 4b 6f 4b 7f 45 5f 51 2d 77 64 36 48 43 6f 32 3b 46 3d 38 66 67 54 5e 39 30 52 75 24 4c 3b 34 7e 2d 50 74 41 4e 43 6e 66 43 7b 7e 0 7b 22 4e 61 6d 65 22 3a 22 54 41 52 4f 20 59 41 4d 41 44 41 22 2c 22 54 65 6c 22 3a 22 30 30 30 2d 30 30 30 30 2d 30 30 30 30 22 7d
3-u{lHctw*ldTA/OeQ|gYC5O|tTcZ2?37}w'6C@4pMuw{12s@|f~kz|<ipH<bA8G>il&S|:lS#Ocxu+MLzbL.Jwe*$2I=|i9sGE#e*j`J4aXd_YllX{<S&-s5fOFSn8}w*Mv??9h@G9R&"[^|ph5JX!nJuACh$VoM.:P!KoKE_Q-wd6HCo2;F=8fgT^90Ru$L;4~-PtANCnfC{~{"Name":"TARO YAMADA","Tel":"000-0000-0000"}
javax.crypto.BadPaddingException: Decryption error
        at java.base/sun.security.rsa.RSAPadding.unpadV15(RSAPadding.java:369)
        at java.base/sun.security.rsa.RSAPadding.unpad(RSAPadding.java:282)
        at java.base/com.sun.crypto.provider.RSACipher.doFinal(RSACipher.java:371)
        at java.base/com.sun.crypto.provider.RSACipher.engineDoFinal(RSACipher.java:405)
        at java.base/javax.crypto.Cipher.doFinal(Cipher.java:2202)
        at com.test.App.decryptInfo(App.java:63)
        at com.test.App.main(App.java:29)
```

- 下のテキスト（正常データ）
```
bytes: 0 2 3e 39 48 52 37 4c 4e 6c 7d 40 75 39 4c 52 2b 71 21 4c 4e 43 23 38 3a 3b 4f 7e 6e 37 31 74 7b 5e 72 25 46 45 50 20 52 58 3a 5e 55 59 70 4a 25 56 4f 64 31 52 49 24 77 48 5b 62 68 3a 3c 4e 53 3f 3c 4b 59 4f 22 21 34 4a 6e 70 71 51 2f 6b 5a 32 5f 51 7a 37 52 32 63 4d 27 70 62 7a 29 31 59 60 40 28 47 3f 3b 2e 61 60 63 70 3e 69 79 52 7f 71 67 67 5b 31 7d 44 43 79 48 64 3d 75 69 46 6a 3e 3a 42 3f 30 2f 36 67 5e 4c 73 5e 59 4e 22 69 68 60 51 3e 6c 6f 38 6f 63 21 3f 76 27 76 50 60 3d 4d 6f 63 40 3e 5f 52 21 78 21 5a 22 5c 67 62 21 69 4f 57 23 2e 45 3e 48 73 2e 7e 4c 43 69 50 69 66 4b 2d 4b 3d 2c 79 6d 4b 23 5d 32 6f 74 68 21 31 0 7b 22 4e 61 6d 65 22 3a 22 54 41 52 4f 20 59 41 4d 41 44 41 22 2c 22 54 65 6c 22 3a 22 30 30 30 2d 30 30 30 30 2d 30 30 30 30 22 7d
>9HR7LNl}@u9LR+q!LNC#8:;O~n71t{^r%FEP RX:^UYpJ%VOd1RI$wH[bh:<NS?<KYO"!4JnpqQ/kZ2_Qz7R2cM'pbz)1Y`@(G?;.a`cp>iyRqgg[1}DCyHd=uiFj>:B?0/66g^Ls^YN"ih`Q>lo8oc!?v'vP`=Moc@>_R!x!Z"\gb!iOW#.E>Hs.~LCiPifK-K=,ymK#]2oth!1{"Name":"TARO YAMADA","Tel":"000-0000-0000"}
{"Name":"TARO YAMADA","Tel":"000-0000-0000"}
```

このように、どちらも `{"Name":"TARO YAMADA","Tel":"000-0000-0000"}` をRSA暗号化したが、上のテキストは先頭のパディングが `0x00 | 0x00 | 0x02` となっており、 `0x00` が1個多いのでパディングエラーとなったことが分かりました。

# 参考資料
- [Java(tm) Platform Standard Edition 8: クラスCipher](https://docs.oracle.com/javase/jp/8/docs/api/javax/crypto/Cipher.html)
- [RSAPadding.java のコード](https://github.com/frohoff/jdk8u-dev-jdk/blob/master/src/share/classes/sun/security/rsa/RSAPadding.java)
- [RSACipher.java のコード](https://github.com/frohoff/jdk8u-jdk/blob/master/src/share/classes/com/sun/crypto/provider/RSACipher.java)