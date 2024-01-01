---
title: "個人取引先無効組織でもエラーにならない、Apexで個人取引先有効組織にしかない項目へアクセスする方法"
emoji: "🏅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["salesforce", "apex", "PersonAccount"]
published: true
published_at: 2024-01-02 09:00
---

# はじめに
個人取引先を有効にした組織では、以下のような個人取引先用の項目にアクセスすることができます。

- IsPersonAccount
  - 取引先レコードが個人取引先か否かを判定する項目
- FirstName, LastName
  - 個人取引先の姓、名

一方、個人取引先が無効の組織でこれらの項目にApexでアクセスしようとすると`No such column 'FirstName' on entity 'Account'`のようにエラーになります。
通常、個人取引先が無効の組織では個人取引先用の項目にアクセスする必要はないので問題にはならないのですが、AppExchangeパッケージのようにどの組織でも使えるApexコードを用意する必要があると問題になります。
例えば、AppExchangeパッケージに以下のようなコードを入れると個人取引先が無効の組織ではエラーになり、インストールが失敗します。

```apex
Account[] accountList = [Select Id, Name, IsPersonAccount from Account Where ...];
for(Account acc : accountList){
    if(acc.IsPersonAccount){ //個人取引先レコードの場合
        ...
    }
    else{ //法人取引先レコードの場合
        ...
    }
}
```

今回はこれを回避し、「個人取引先が無効の組織でもエラーにならない、個人取引先用項目にアクセス可能なApexコード」を書いてみます。

# 考え方
上述の通り、`acc.IsPersonAccount`のような形で項目にアクセスすると、個人取引先が無効の組織では「その項目はありません」エラーになります。
これを回避する方法として、[SObject クラスのget(fieldName)メソッド](https://developer.salesforce.com/docs/atlas.ja-jp.apexcode.meta/apexcode/apex_methods_system_sobject.htm#apex_System_SObject_get)を使います。
`get`メソッドの`fieldName`引数は文字列を使用するため、`get('IsPersonAccount')`を個人取引先無効の組織で使用しても、実行しなければエラーにはなりません。

同様にSOQLについても[Database クラスのqueryメソッド](https://developer.salesforce.com/docs/atlas.ja-jp.apexcode.meta/apexcode/apex_methods_system_database.htm#apex_System_Database_query)を使用し、クエリ文を文字列とすることで、IsPersonAccount、FirstName、LastNameをSelect文で取得するコードを記載しても個人取引先無効の組織でエラーになりません。

# 個人取引先無効組織でもエラーにならないコード
今回書いたコードはこのような形になりました。

- コード

```apex
public class GetPersonAccountInfo {
    public static void getName(String accId){
        String[] fieldsArray = new String[]{
            'Id', 'Name', 'FirstName', 'LastName', 'IsPersonAccount'
        };
        String fields = String.join(fieldsArray, ',');
        String query = String.format('Select {0} From Account Where Id = \'\'{1}\'\'',
                                        new String[]{String.escapeSingleQuotes(fields),
                                        String.escapeSingleQuotes(accId)});
        try{
            List<Account> accList = Database.query(query);
            for(Account acc: accList){
                Boolean isPersonAccount = Boolean.valueOf(acc.get('IsPersonAccount'));
                if(isPersonAccount){
                    System.debug('FirstName: ' + acc.get('FirstName'));
                    System.debug('LastName: ' + acc.get('LastName'));
                }
                else{
                    System.debug('Name: ' + acc.get('Name'));
                }
            }
        }
        catch(Exception e){
            System.debug(e.getTypeName()+ ':' +e.getMessage());
        }
    }
}
```

- 呼び出し例

```apex
GetPersonAccountInfo.getName('001aaBBBBBcccccAAA');
```

# おわりに
上記`GetPersonAccountInfo.getName()`メソッドはあくまで「個人取引先が無効でも保存できるApexコード」であって、個人取引先が無効な組織で実行すれば当然Exceptionが発生します。
実際のロジックでは[こちらのブログ](https://tyoshikawa1106.hatenablog.com/entry/2019/08/16/080827)で紹介されているように`Schema.sObjectType.Account.fields.getMap().containsKey('isPersonAccount')`等で組織として個人取引先が有効にされているか否かを判定したうえで、個人取引先が有効な組織の場合のみに`GetPersonAccountInfo.getName()`メソッドを実行する必要があります。

# 参考文献
- [Check if Person Account is enabled from APEX - wedgecommerce](https://wedgecommerce.com/check-person-account-enabled-apex/)