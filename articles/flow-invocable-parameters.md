---
title: "フローでApexアクションとして使用するApexの引数指定"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flow", "salesforce", "apex"]
published: true
published_at: 2023-04-27 07:00
---

# はじめに

フローではApexアクション要素としてApexクラスを呼び出すことができます。
その際にフロー変数をApex側の変数として渡したりApexの返却値をフローで受け取ることができますが、Apex側の変数の型を正しく指定しないとフロー側で入力、出力値の指定ができなくなったりします。
ここの指定に悩んだので、メモとして調べた内容を記載します。

# 入力
以下、レコードID（フロー側変数では「テキスト」）をフローからApexに渡すときの変数の型です。

| 使用アノテーション | フロー側の変数の型 | Apex変数の型 |
| --- | --- | --- |
| `@InvocableMethod` | 単一変数 | `List<Id>` |
| `@InvocableMethod` | コレクション変数 | `List<List<Id>>` |
| `@InvocableVariable` | 単一変数 | `Id` |
| `@InvocableVariable` | コレクション変数 | `List<Id>` |

文章で書くと、以下のイメージです。

- `@InvocableMethod` アノテーションを使って、メソッドの引数で変数を指定する場合は、Apex変数の型はListでくくる。
- `@InvocableVariable` アノテーションを使って、別クラスで変数を指定する場合は、Apex変数の型は想像通り（単一変数ならListなし、コレクション変数ならListあり）。
    - `@InvocableMethod` 側は、InvocableVariable用別クラスをListでくくった型とする。

## 実例
### `@InvocableMethod` アノテーションのみの場合
`@InvocableMethod` アノテーションの場合は単一変数、コレクション変数どちらでも `get` メソッドで欲しいデータを取得します。

#### 単一変数を渡す

```apex
public class flowToApexSample(){
    @InvocableMethod(label='flowToApexSample')
    public static List<Id> sample1(List<Id> requests){
        //単一変数のIdをrequestIdに渡す
        Id requestId = requests.get(0);
        ...
    }
}
```

#### コレクション変数を渡す

```apex
public class flowToApexSample(){
    @InvocableMethod(label='flowToApexSample')
    public static List<Id> sample1(List<List<Id>> requests){
        //コレクション変数内のIdをrequestIdsに渡す
        List<Id> requestIds = requests.get(0);
        ...
    }
}
```

### `@InvocableVariable` アノテーション用別クラスを作成する場合

```apex
public class flowToApexSample(){
    //入力変数用のクラスを作り、変数を入れる
    public class sampleInputInfo(){
        //単一変数&必須
        @InvocableVariable(required=true)
        public Id contextId;

        //コレクション変数&任意
        @InvocableVariable
        public List<Id> selectedIds;
    }

    @InvocableMethod(label='flowToApexSample')
    public static List<Id> sample1(List<sampleInputInfo> infos){
        Id contextId = infos.get(0).contextId;
        List<Id> selectedIds = infos.get(0).selectedIds;
        ...
    }
}
```

# 出力

出力（Apex→フロー）も、入力と同様の変数の型を指定した変数をreturn文で指定すればフローに渡せます。

## 実例
### `@InvocableMethod` アノテーションのみの場合
#### 単一変数を返す

```apex
public class flowToApexSample(){
    @InvocableMethod(label='flowToApexSample')
    public static List<Id> sample1(List<Id> requests){
        ...
        List<Id> returnId = new List<Id>();
        returnId.add(/*ここに返したいId変数を指定*/);
        return returnId;
    }
}
```

#### コレクション変数を返す

```apex
public class flowToApexSample(){
    @InvocableMethod(label='flowToApexSample')
    public static List<List<Id>> sample1(List<List<Id>> requests){
        ...
        List<List<Id>> returnIds = new List<List<Id>>();
        returnIds.add(/*ここに返したいList<Id>型の変数を指定*/);
        return returnIds;
    }
}
```

### `@InvocableVariable` アノテーション用別クラスを作成する場合

```apex
public class flowToApexSample(){
    //出力変数用のクラスを作り、変数を入れる
    public class sampleOutputInfo(){
        //単一変数
        @InvocableVariable
        public Id contextId;

        //コレクション変数
        @InvocableVariable
        public List<Id> selectedIds;
    }

    @InvocableMethod(label='flowToApexSample')
    public static List<sampleOutputInfo> sample1(List<Id> requests){
        ...
        //sampleIdはId型、sampleIdsはList<Id>型の変数
        sampleOutputInfo returnInfo = new sampleOutputInfo();
        returnInfo.contextId = sampleId;
        returnInfo.selectedIds = sampleIds;

        List<sampleOutputInfo> returnInfos = new List<sampleOutputInfo>();
        returnInfos.add(returnInfo);
        return returnInfos;
    }
}
```

# 参考資料

- [Apex開発者ガイド: InvocableVariable アノテーション](https://developer.salesforce.com/docs/atlas.ja-jp.apexcode.meta/apexcode/apex_classes_annotation_InvocableVariable.htm)
- [Invocable Methods: How to Send Data Between Flow and Apex](https://unhandledsunshine.com/2021/08/12/invocable-methods-how-to-send-data-between-flow-and-apex/)
