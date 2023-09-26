---
title: "カスタムメタデータを用いたApexのテスト方法"
emoji: "💯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["salesforce", "apex", "testvisible"]
published: true
published_at: 2023-09-24 01:00
---

# はじめに
Salesforceの[カスタムメタデータ](https://help.salesforce.com/s/articleView?id=sf.custommetadatatypes_about.htm&type=5&language=ja)は、独自のメタデータ型で設定内容を定義し、ユーザ側で内容をオブジェクトのレコードのように設定できる機能です。
パッケージの設定をこの機能で行ったりもしますが、Apexコードに必須のテストコードを書く際には、通常のオブジェクトレコードのように「テスト用のレコードをテストクラス内で作成する」ことはできず、通常は実際に存在するカスタムメタデータレコードそのものの値を使用することになります。
つまり複数の処理パターンを実装したApexクラスであっても1パターンしかテストできずテストカバレッジが稼げないため、相性が悪い機能となってしまっています。
ここではカスタムメタデータを用いたApexコードで複数の処理パターンをテストするために調べた方法を記載します。

# テスト方法
## 対象のApexクラスコード
今回はカスタムメタデータオブジェクト:Sample_Setting__mdt から apiKey__c(テキスト項目) と flag__c(チェックボックス項目) を取り出す以下のようなコードを考えます。

```apex:Sample.cls
public class Sample {
    private static Sample_Setting__mdt sampleMdt = Sample_Setting__mdt.getInstance('Settings');
    private static String apiKey = sampleMdt.apiKey__c;
    private static boolean flag = sampleMdt.flag__c;
    //以下略
}
```

## DAO(Data Access Object)
上のコードではSample_Setting__mdtオブジェクトのSettingsレコードに直接アクセスしていますが、SampleSettingDAOクラスを介する形とします。
この時、SampleSettingDAOクラスには[setプロパティ](https://developer.salesforce.com/docs/atlas.ja-jp.apexcode.meta/apexcode/apex_classes_properties.htm)を用意します。

```apex:SampleSettingDAO.cls
public class SampleSettingDAO {
    public static Sample_Setting__mdt sampleSetting {
        get{
            if(sampleSetting == null) {
                sampleSetting = Sample_Setting__mdt.getInstance('Settings');
            }
            return sampleSetting;
        }
        set;
    }
}
```

```apex:Sample.cls
public class Sample {
    //SampleSettingDAOクラスを介してアクセス
    private static Sample_Setting__mdt sampleMdt = SampleSettingDAO.sampleSetting;
    private static String apiKey = sampleMdt.apiKey__c;
    private static boolean flag = sampleMdt.flag__c;
    //以下略
}
```

```apex:SampleTest.cls
@isTest
private class SampleTest {
    @isTest
    static void testPattern1(){
        //テスト用のsampleSettingを用意する
        SampleSettingDAO.sampleSetting = new Sample_Setting__mdt(apiKey__c='a', flag__c=false);
        Sample s = new Sample();
        //以下略
    }
}
```

これで複数パターンのテストは可能ですが、実際の処理には不要なのにテストのためだけにDAOクラスを用意するのは非常に面倒です。

## `@testVisible` アノテーション
[こちらの方の投稿](https://twitter.com/tarot/status/1691382054628581376)を参考にさせていただきました。
カスタムメタデータ型から取得した値を保存する変数に `@testVisible` アノテーションを付与し、テストクラス側では変数の値そのものを書き換える方法です。

```apex:Sample.cls
public class Sample {
    private static Sample_Setting__mdt sampleMdt = Sample_Setting__mdt.getInstance('Settings');
    //@testVisible アノテーションを付与
    @testVisible private static String apiKey = sampleMdt.apiKey__c;
    @testVisible private static boolean flag = sampleMdt.flag__c;
    //以下略
}
```

```apex:SampleTest.cls
@isTest
private class SampleTest {
    @isTest
    static void testPattern1(){
        Sample s = new Sample();
        Sample.apiKey = 'a';
        Sample.flag = false;
        //以下略
    }
}
```

これでも複数パターンのテストは可能です。

# 参考資料
- [カスタムメタデータのテストについて調べてみた - DeveloperIO](https://dev.classmethod.jp/articles/custom-meta-data-test/)
  - カスタムメタデータのテスト方法について、使用できない方法含め丁寧に解説されています。ただこの投稿ではDAOクラスの利用しか説明がないため、今回これを作成しました。
