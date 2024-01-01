---
title: "sObject変数から項目を削減する"
emoji: "🗑️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["salesforce", "apex"]
published: true
published_at: 2024-01-02 09:00
---

# はじめに
こんなロジックを組みたいとします。
```apex
Account acct = [Select Id, Name, AnnualRevenue From Account Where Id = '001xxyyyyyzzzzz'][0];

//SOQLで取得した項目を使って何らかの処理を行う

//適当な項目を更新
acct.AnnualRevenue = ...;

//更新を反映
update acct;
```

しかし、個人取引先レコードに対してこの処理を行うと、以下のようにSystem.DmlExceptionが発生してエラーになります。

```
System.DmlException : Update failed. First exception on row 0 with id 0015g00001RGaPtAAL; first error: INVALID_FIELD_FOR_INSERT_UPDATE, Unable to create/update fields: Name. Please check the security settings of this field and verify that it is read/write for your profile or permission set.: [Name]
```

エラーを読むと、「Name項目に対する更新権限がないのでエラー」のようです。
このロジックではName項目に対して更新しないので、以下のようにName項目を除去したsObject変数を用いてupdateをかければ回避できます。

```apex
Account acct = [Select Id, Name, AnnualRevenue From Account Where Id = '001xxyyyyyzzzzz'][0];

//SOQLで取得した項目を使って何らかの処理を行う

//適当な項目を更新
acct.AnnualRevenue = ...;

//更新を反映
Account updateAcct = new Account();
updateAcct.Id = acct.Id;
updateAcct.AnnualRevenue = acct.AnnualRevenue;
update updateAcct;
```

しかし、update対象項目の数が多いと面倒です。
単純にsObject変数から特定項目を除去できるメソッドがあればよかったのですが、[Apex開発者ガイド:SObject クラス](https://developer.salesforce.com/docs/atlas.ja-jp.apexcode.meta/apexcode/apex_methods_system_sobject.htm)を見る限りなさそうです。
ということで、sObject変数から特定の項目を削減するメソッドを作成してみました。

# sObject変数から特定の項目を削減するメソッド
## 実際のコード
どのオブジェクトでも使用できるようsObject型で作成してみました。

```apex
public class SObjectUtil {
    public static List<sObject> RemovalFieldsFromSObject(List<sObject> sObjectList, List<String> removalFields){
        List<sObject> resultList = new List<sObject>(); //return用変数
        for(sObject record : sObjectList){
            //record変数のsObjectTypeを取り出す
            schema.sObjectType recordObjectType = record.getSObjectType();
            //record変数と同じsObjectTypeのsObject変数を作成
            sObject s = recordObjectType.newSObject();

            //record変数に含まれる項目と項目値をfieldsToValue変数に出力
            Map<String, Object> fieldsToValue = record.getPopulatedFieldsAsMap();
            for(String fieldName : fieldsToValue.keySet()){
                if(!removalFields.contains(fieldName)){
                    //項目名がremovalFieldsに含まれない場合のみ追加
                    s.put(fieldName, fieldsToValue.get(fieldName));
                }
            }
            resultList.add(s);
        }
        return resultList;
    }
}
```

## 実行サンプル

- 実行コード

```apex
Account[] acc = [Select Id, Name From Account Where Id = '001aaBBBBBcccccDDD'];
List<String> removalFields = new List<String>{'Name'};
System.debug(acc[0].getPopulatedFieldsAsMap());
Account[] acc2 = SObjectUtil.RemovalFieldsFromSObject(acc, removalFields);
System.debug(acc2[0].getPopulatedFieldsAsMap());
```

- 実行ログ

```text
//RemovalFieldsFromSObjectメソッド処理前
{Id=001aaBBBBBcccccDDD, Name=sample test}

//RemovalFieldsFromSObjectメソッド処理後
{Id=001aaBBBBBcccccDDD}
```

期待通り、RemovalFieldsFromSObjectメソッド処理後ではName項目が除去されました。

# 終わりに
初めは最初のエラー発生箇所に数式を使おうと思ったんですが、数式項目を含むsObject変数に対しupdateをかけても普通に更新成功しました。
なのであまり利用用途はないかもしれませんが、sObject変数から項目を削減するにはどうするんだろう、と思ったことはこれ以前にもあったので今回書いてみました。

# 参考文献
- [Remove a field from an sObject object in Apex - Salesforce Stack Exchange](https://salesforce.stackexchange.com/questions/199784/remove-a-field-from-an-sobject-object-in-apex)