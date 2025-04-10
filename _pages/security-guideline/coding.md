---
title: EC-CUBE 4 セキュリティガイドライン
keywords: coding
tags: [security-guideline, coding]
permalink: security-guideline/coding
folder: security-guideline
---
EC-CUBE 4 セキュリティガイドライン

# 概要

## 目的
本ガイドラインは、EC-CUBEの開発および、EC-CUBEを用いたカスタマイズにおける、開発者向けのガイドラインです。
本ガイドラインにそって開発することで、脆弱性への対策を実装段階で組み込み、セキュリティの品質を維持・向上することを目的としています。

## 対象読者
主にEC-CUBEの開発および、EC-CUBEを用いたカスタマイズを行う開発者を想定しています。

## 記載範囲
EC-CUBEの開発およびEC-CUBEを用いたカスタマイズに対し、主に実装フェーズでのコーディングにおける注意事項を記載しています。
EC-CUBEの設置・保守運用に関する記載は、含まれておりません。

# 脆弱性予防のための注意事項

[安全なウェブサイトの作り方](https://www.ipa.go.jp/security/vuln/websecurity.html) に記載の、「ウェブアプリケーションのセキュリティ実装」に基づき、各脆弱性を予防するためにEC-CUBEのコーディング時に注意すべき事項を記載します。
脆弱性の詳細な解説やWebアプリケーション開発での一般的な注意事項については、「安全なウェブサイトの作り方」の原本を参照してください。
※原本参照箇所については、引用の形式で記載しております。

## SQLインジェクション

### SQLインジェクション

#### [根本的解決] SQL文の組み立ては全てプレースホルダで実装する
> SQLには通常、プレースホルダを用いてSQL文を組み立てる仕組みがあります。SQL文の雛形の中に変数の場所を示す記号（プレースホルダ）を置いて、後に、そこに実際の値を機械的な処理で割り当てるものです。ウェブアプリケーションで直接、文字列連結処理によってSQL文を組み立てる方法に比べて、プレースホルダでは、機械的な処理でSQL文が組み立てられるので、SQLインジェクションの脆弱性を解消できます。  
> プレースホルダに実際の値を割り当てる処理をバインドと呼びます。バインドの方式には、プレースホルダのままSQL文をコンパイルしておき、データベースエンジン側で値を割り当てる方式（静的プレースホルダ）と、アプリケーション側のデータベース接続ライブラリ内で値をエスケープ処理してプレースホルダにはめ込む方式（動的プレースホルダ）があります。静的プレースホルダは、SQLのISO/JIS規格では、準備された文(Prepared Statement)と呼ばれます。  
> どちらを用いてもSQLインジェクション脆弱性を解消できますが、原理的にSQLインジェクション脆弱性の可能性がなくなるという点で、静的プレースホルダの方が優ります。詳しくは[「安全なSQLの呼び出し方」](https://www.ipa.go.jp/files/000017320.pdf)のプレースホルダの項（3.2節）を参照してください。

EC-CUBEでは、O/R Mapperを利用しておりSQLを直接記述している箇所はないため問題ありません。  
特別な事情がない限りは、O/RMapperの使用を推奨しております。  
なお、カスタマイズ時以下2点を注意してください。

##### 1. QueryBuilderを利用する際の注意


* 誤った例：

```php
$qb = $this->createQueryBuilder('p')
    ->where('p.price > '. $price)
    ->orderBy('p.price', 'ASC');
```

※`$price` の中身に`1 or 1=1` といったような文字が入り込むと、すべてのレコードが返ってくることになります。

* 推奨例：

```php
$qb = $this->createQueryBuilder('p')
    ->where('p.price > :price')
    ->setParameter('price', $price)
    ->orderBy('p.price', 'ASC');
```

※`setParameter` を使用することで、SQLインジェクションのリスクを減らすことが出来ます。

##### 2. dbal利用する際の注意

以下のような構文は使用しないようにお願いします。
`$price` の中身に`1 or 1=1` といったような文字が入り込むと、すべてのレコードが返ってくることになります。
これは意図しない結果となるかつ、中身によっては他のテーブルのデータの閲覧・変更・削除のリスクがあります。

* 誤った例：

```php
$conn = $this->getEntityManager()->getConnection();

$sql = "
    SELECT * FROM product p
    WHERE p.price > $price
    ORDER BY p.price ASC
    ";
$stmt = $conn->prepare($sql);
$resultSet = $stmt->executeQuery();

// returns an array of arrays (i.e. a raw data set)
return $resultSet->fetchAllAssociative();
```

* 推奨された例

```php
$conn = $this->getEntityManager()->getConnection();

$sql = '
    SELECT * FROM product p
    WHERE p.price > :price
    ORDER BY p.price ASC
    ';
$stmt = $conn->prepare($sql);
$resultSet = $stmt->executeQuery(['price' => $price]);
```

※O/R Mapperについては、利用方法を誤ると適切な対策が行われれない場合がございます。利用方法について注意してください。

#### [根本的解決] SQL文の組み立てを文字列連結により行う場合は、エスケープ処理等を行うデータベースエンジンのAPIを用いて、SQL文のリテラルを正しく構成する
> SQL文の組み立てを文字列連結により行う場合は、SQL文中で可変となる値をリテラル（定数）の形で埋め込みます。値を文字列型として埋め込む場合は、値をシングルクォートで囲んで記述しますが、その際に文字列リテラル内で特別な意味を持つ記号文字をエスケープ処理します（たとえば、「'」→「''」、「\」→「\\」等）。値を数値型として埋め込む場合は、数値リテラルであることを確実にする処理（数値型へのキャスト等）を行います。  
> こうした処理で具体的に何をすべきかは、データベースエンジンの種類や設定によって異なるため、それにあわせた実装が必要です。データベースエンジンによっては、リテラルを文字列として生成する専用のAPIを提供しているものがありますので、それを利用することをお勧めします。詳しくは、[「安全なSQLの呼び出し方」](https://www.ipa.go.jp/files/000017320.pdf)の4.1節を参照してください。なお、この処理は、外部からの入力の影響を受ける値のみに限定して行うのではなく、SQL文を構成する全てのリテラル生成に対して行ってください。

EC-CUBEでは、O/R Mapperを利用しておりSQLを直接記述している箇所はないため問題ありません。
なお、カスタマイズ時以下2点を注意してください。

##### 1. QueryBuilderを利用する際の注意

基本的にSymfonyで実装する場合は、前述の「SQL文の組み立ては全てプレースホルダで実装する」と同じ注意点です。
加えて、処理をfunctionでラップすることで、型を限定することが出来ます。

```php
public function findAllGreaterThanPrice(int $price, bool $includeUnavailableProducts = false): array
{
    // automatically knows to select Products
    // the "p" is an alias you'll use in the rest of the query
    $qb = $this->createQueryBuilder('p')
        ->where('p.price > :price')
        ->setParameter('price', $price)
        ->orderBy('p.price', 'ASC');

    if (!$includeUnavailableProducts) {
        $qb->andWhere('p.available = TRUE');
    }

    $query = $qb->getQuery();

    return $query->execute();
}
```

##### 2. dbal利用する際の注意

こちらもfunctionでラップすることで、型を限定することが可能です。

```php
public function findAllGreaterThanPrice(int $price): array
{
    $conn = $this->getEntityManager()->getConnection();

    $sql = '
        SELECT * FROM product p
        WHERE p.price > :price
        ORDER BY p.price ASC
        ';
    $stmt = $conn->prepare($sql);
    $resultSet = $stmt->executeQuery(['price' => $price]);

    return $resultSet->fetchAllAssociative();
}
```

※O/R Mapperについては、利用方法を誤ると適切な対策が行われれない場合がございます。利用方法について注意してください。

#### [根本的解決] ウェブアプリケーションに渡されるパラメータにSQL文を直接指定しない
> EC-CUBEに関わらず、hiddenパラメータ等にSQL文をそのまま指定するという事例が散見されますので、避けるべき実装として紹介します。ウェブアプリケーションに渡されるパラメータにSQL文を直接指定する実装は、そのパラメータ値の改変により、データベースの不正利用につながる可能性があります。

カスタマイズ時、直接SQL実行できるプラグインを作成することも可能となるため上記点に注意してください。

#### [保険的対策] エラーメッセージをそのままブラウザに表示しない
> エラーメッセージの内容に、データベースの種類やエラーの原因、実行エラーを起こしたSQL文等の情報が含まれる場合、これらはSQLインジェクション攻撃につながる有用な情報となりえます。また、エラーメッセージは、攻撃の手がかりを与えるだけでなく、実際に攻撃された結果を表示する情報源として悪用される場合があります。データベースに関連するエラーメッセージは、利用者のブラウザ上に表示させないことをお勧めします。

EC-CUBEでは、エラーメッセージはdevモードの場合のみ表示しているため問題ありません。

#### [保険的対策] データベースアカウントに適切な権限を与える
> ウェブアプリケーションがデータベースに接続する際に使用するアカウントの権限が必要以上に高い場合、攻撃による被害が深刻化する恐れがあります。ウェブアプリケーションからデータベースに渡す命令文を洗い出し、その命令文の実行に必要な最小限の権限をデータベースアカウントに与えてください。

EC-CUBEでは、サーバの設定で対応を行ってください。

[](## OSコマンド・インジェクション)

## パス名パラメータの未チェック／ディレクトリ・トラバーサル

### ディレクトリ・トラバーサル

#### [根本的解決] 外部からのパラメータでウェブサーバ内のファイル名を直接指定する実装を避ける
> 外部からのパラメータでウェブサーバ内のファイル名を直接指定する実装では、そのパラメータが改変され、任意のファイル名を指定されることにより公開を想定しないファイルが外部から閲覧される可能性があります。たとえば、HTML中のhiddenパラメータでウェブサーバ内のファイル名を指定し、そのファイルをウェブページのテンプレートとして使用する実装では、そのパラメータが改変されることで、任意のファイルをウェブページとして出力してしまう等の可能性があげられます。外部からのパラメータでウェブサーバ内のファイル名を直接指定する実装が本当に必要か、他の処理方法で代替できないか等、仕様や設計から見直すことをお勧めします。

EC-CUBEでは、ファイル管理機能でファイル名の指定を行っており絶対パスではなく`user_data`からの相対パスであるが、従来ディレクトリトラバーサルを受けやすいため、外部からのファイル名直接指定を避ける方法として、以下いずれかの対策を行いカスタマイズを行ってください。
```
・ファイル名を固定にする
・ファイル名をセッション変数に保持する
・ファイル名を直接指定するのではなく番号などで間接的に指定する
```
※参考（ディレクトリ・トラバーサルの対策）：「体系的に学ぶ安全なWebアプリケーションの作り方 第2版」（徳丸浩 著）P.285

#### [根本的解決] ファイルを開く際は、固定のディレクトリを指定し、かつファイル名にディレクトリ名が含まれないようにする
> たとえば、カレントディレクトリ上のファイル「filename」を開くつもりで、`open(filename) `の形式でコーディングしている場合、`open(filename) `のfilenameに絶対パス名が渡されることにより、任意ディレクトリのファイルが開いてしまう可能性があります。この絶対パス名による指定を回避する方法として、あらかじめ固定のディレクトリ「dirname」を指定し、`open(dirname+filename) `のような形でコーディングする方法があります。しかし、これだけでは、「../」等を使用したディレクトリ・トラバーサル攻撃を回避できません。これを回避するために、`basename() `等の、パス名からファイル名のみを取り出すAPIを利用して、`open(dirname+basename(filename)) `のような形でコーディングして、filenameに与えられたパス名からディレクトリ名を取り除くようにします。

EC-CUBEでは、ファイル管理機能でファイル名の指定を行っており絶対パスではなく`user_data`からの相対パスであるが、従来ディレクトリトラバーサルを受けやすいため、外部からのファイル名直接指定を避ける方法として、以下いずれかの対策を行いカスタマイズを行ってください。
※相対パスを使わないといけない状況では、ファイル名のみを抜き出す対策は使えないため代替として、パスの正規化による対策を行ってください。
```
1．phpのrealpath() などを用い、パスを正規化する。
2．path名先頭一致の検査で、許可されたディレクトリ以下のファイルであることを確認する。
```

* 誤った例

```php
$fileName = $request->getParameter('file_path');
$dirName = "/path/to/directory/";

$handle = open($dirName . $fileName, "r");
```

* 推奨された例

```php
$fileName = $request->getParameter('file_path');
$dirName = "/path/to/directory/";

$handle = open($dirName . basename($fileName), "r");
```

もしくは

```php
$fileName = $request->getParameter('file_path');
$dirName = "/path/to/directory/";

$handle = open($dirName . realpath($fileName), "r");
```

参考（phpのrealpath関数）：[https://www.php.net/manual/ja/function.realpath.php](https://www.php.net/manual/ja/function.realpath.php)


#### [保険的対策] ウェブサーバ内のファイルへのアクセス権限の設定を正しく管理する
> ウェブサーバ内に保管しているファイルへのアクセス権限が正しく管理されていれば、ウェブアプリケーションが任意ディレクトリのファイルを開く処理を実行しようとしても、ウェブサーバ側の機能でそのアクセスを拒否できる場合があります。

```
drwxr-xr-x 1 apache apache 0 Aug 8 13:01 html　←こちらのディレクトリはアクセスできる
drwxr-x--- 1  admin  admin 0 Aug 8 13:01 others ←こちらのディレクトリはアクセスできない
```

#### [保険的対策] ファイル名のチェックを行う
> ファイル名を指定した入力パラメータの値から、「/」、「../」、「..\」等、OSのパス名解釈でディレクトリを指定できる文字列を検出した場合は、処理を中止します。ただし、URLのデコード処理を行っている場合は、URLエンコードした「%2F」、「..%2F」、「..%5C」、さらに二重エンコードした「%252F」、「..%252F」、「..%255C」がファイル指定の入力値として有効な文字列となる場合があります。チェックを行うタイミングに注意してください。

EC-CUBEでは、ファイル名のバリデーションを行っているため問題ありません。

[](## セッション管理の不備)
## クロスサイト・スクリプティング

### HTMLテキストの入力を許可しない場合の対策

#### [根本的解決] ウェブページに出力する全ての要素に対して、エスケープ処理を施す

EC-CUBEのtwigテンプレートでの出力は、標準ではHTMLエスケープが行われます。
rawフィルタ等を用い、意図的にエスケープを解除することは可能ですが、その際、外部からの入力値が含まれていると、攻撃手段として利用される可能性があります。

```
{{ "{% raw " }}%}{{"{{ variable|raw " }}}}{{ "{% endraw " }}%}
```

rawフィルタ以外でもHTMLエスケープの解除は可能です。
以下のtwigフィルタやphp関数を利用する場合も注意してください。

```
{{ "{% apply raw " }}%}

{{"{{ variable " }}}}

{{ "{% endapply " }}%}
```

```
{{ "{% autoescape false " }}%}

{{"{{ variable " }}}}

{{ "{% endautoescape " }}%}
```

```
htmlspecialchars_decode
html_entity_decode
strip_tags
```

##### HTMLエスケープを解除する場合は、以下の点に留意してください。

- 必要がないにもかかわらず、HTMLエスケープを解除していないか
- 外部からの入力値をそのまま出力していないか
- 外部からの入力値を出力している場合、意図したものかどうか
- 外部からの入力値を出力している場合、不正な入力が入り込む余地はないか

特に、管理画面でエスケープを解除する場合は、**値にユーザ入力が含まれないか**どうかを注意してください。

#### [根本的解決] URLを出力するときは、「http://」や「https://」で始まるURLのみを許可する

URLのバリデーションは、Symfony標準のUrlValidatorを利用するようにし、http://およびhttps://で始まるURLのみ許容するようにしてください。

```php
class XxxType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder
            ->add('url', TextType::class, [
                'constraints' => [
                    new Assert\Url(),
                ],
            ]);
```

#### [根本的解決] `<script>...</script>` 要素の内容を動的に生成しない

> ウェブページに出力する`<script>...</script>`要素の内容が、外部からの入力に依存する形で動的に生成される場合、任意のスクリプトが埋め込まれてしまう可能性があります。危険なスクリプトだけを排除する方法も考えられますが、危険なスクリプトであることを確実に判断することは難しいため、`<script>...</script>`要素の内容を動的に生成する仕様は、避けることをお勧めします。

記述は避けるべき記述パターン
```
<script>{{"{{ script_text|raw " }}}}</script>
```

対策としては、以下をおすすめします。
・script要素の外部でカスタムデータ属性にてパラメータを定義して、JavaScriptから参照する
　
管理画面のテンプレートでは以下のような使い方をしております。
```html
<script src="{{ "{{ asset('assets/js/layout_design.js', 'admin') " }}}}"></script>
```

* また、「動的生成をしない」という対策に反することになりますが、以下のようなエスケープにより対策を行うことも可能です。ただし、手順が複雑で対策漏れが発生する可能性があるため、おすすめはしません。
※script要素内のJavaScript動的生成時の対策

1. JavaScriptの文法から、引用符（「"」と「'」）と「\」や改行をエスケープする
2. 1の結果に「`</script>`」という文字列が出現しないようにする


#### [根本的解決] Javascriptでユーザーが入力した値を利用する際はエスケープ処理を実施する

ユーザーがフォーム等から入力した値をJavascriptから使用する際、安易にそのまま使用するのではなく  
必ずエスケープ処理を挟むようにして下さい。

##### Twigテンプレート側の処理でエスケープする例

```javascript
var message = {{"'{{ form.value|escape('js')" }}}}';
```

`escape('js')`というTwig関数を用いることで、値をエスケープ処理した状態で使用することが可能です。

##### jQueryなどのJSライブラリ側でエスケープする場合

* 誤った例

```javascript
$('#modal').html(this);
```

* 推奨例

```javascript
$('#modal').text(this);
```

※HTMLをそのまま使用する処理を書くのではなく、エスケープした状態で使用する処理に変更する

#### [根本的解決] スタイルシートを任意のサイトから取り込めるようにしない
> スタイルシートには、expression() 等を利用してスクリプトを記述することができます。 外部ライブラリを使用する場合は十分に確認を行い、信頼できないものは利用を避けてください。

使用する場合は、`integrity`を指定し、改ざんの確認を行うようにしてください。

```html
<link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/3.4.1/css/bootstrap.min.css" integrity="sha384-HSMxcRTRxnN+Bdg0JdbxYKrThecOKuH5zCYotlSAcp1+c8xmyTe9GYg1l9a69psu" crossorigin="anonymous">
```

#### [保険的対策] 入力値の内容チェックを行う
> 入力値すべてについて、ウェブアプリケーションの仕様に沿うものかどうかを確認する処理を実装し、仕様に合わない値を入力された場合は処理を先に進めず、再入力を求めるようにする対策方法です。ただし、この対策が有効となるのは限定的です。例えば、アプリケーションの要求する仕様が幅広い文字種の入力を許すものである場合には対策にならないため、この方法に頼ることはお勧めできません。

カスタマイズ時、入力値はFormTypeを用いてバリデーションし、安易に`$request`を使わないようにしてください。

なお、EC-CUBEではフォームの入力項目を各モデルごとのhogehogeType.phpというものに定義をしています。
その中で値のバリデーションが実施できていれば問題ありません。
```php
$builder->add('sample_form_name', TextType::class, [
        'constraints' => [
            new Assert\NotBlank(),
            new Assert\Length(['max' => 20]),
        ],
        'data' => $postedValue
    ]);
```


### HTMLテキストの入力を許可する場合の対策

#### [根本的解決] 入力されたHTMLテキストから構文解析木を作成し、スクリプトを含まない必要な要素のみを抽出する
> 入力されたHTMLテキストに対して構文解析を行い、「ホワイトリスト方式」で許可する要素のみを抽出します。ただし、これには複雑なコーディングが要求され、処理に負荷がかかるといった影響もあるため、実装には十分な検討が必要です。

ただし、EC-CUBE4.1以降、フロントからの入力値はhtmlテキストはエスケープもしくは無害文字列への置換を行っており、アプリケーション全体に適用される（プラグインやカスタマイズのコードにも適用される）ため問題ありません。

なお、当該処理を独自実装するのではなく、html purifier等の広く使われているライブラリを利用することが望ましいです。
※html purifierの利用にあたっては以下のような議論があり、今後推奨外となる可能性があります。
[https://github.com/pkp/pkp-lib/issues/7916](https://github.com/pkp/pkp-lib/issues/7916)

#### [保険的対策] 入力されたHTMLテキストから、スクリプトに該当する文字列を排除する。

> 入力されたHTMLテキストに含まれる、スクリプトに該当する文字列を抽出し、排除してください。抽出した文字列の排除方法には、無害な文字列へ置換することをお勧めします。たとえば、`「<script>」や「javascript:」`を無害な文字列へ置換する場合、`「<xscript>」「xjavascript:」`のように、その文字列に適当な文字を付加します。他の排除方法として、文字列の削除が挙げられますが、削除した結果が危険な文字列を形成してしまう可能性（*）があるため、お勧めできません。

なお、この対策は、危険な文字列を完全に抽出することが難しいという問題があります。ウェブブラウザによっては、`「java&##09;script:」や「java(改行コード)script:」`等の文字列を`「javascript:」`と解釈してしまうため、単純なパターンマッチングでは危険な文字列を抽出することができません。そのため、このような「ブラックリスト方式」による対策のみに頼ることはお勧めできません。

* 例

以下のようなHTMLは、ブラウザによっては同じ用に解釈される可能性があります。

```html
<a href="javascript:exec()"></a>

<a href="java
script:exec()"></a>

<a href="java&##09;script:exec()"></a>
```

ただし、EC-CUBE4.1以降、フロントからの入力値は、htmlテキストはエスケープもしくは無害文字列への置換が行われておりおり、アプリケーション全体に適用される（プラグインやカスタマイズのコードにも適用される）ため問題ありません。

*[3.5.3のよくある失敗例2を参照](https://www.ipa.go.jp/files/000017316.pdf)

### 全てのウェブアプリケーションに共通の対策

#### [根本的解決] HTTPレスポンスヘッダのContent-Typeフィールドに文字コード（charset）を指定する

> HTTPのレスポンスヘッダのContent-Typeフィールドには、`「Content-Type: text/html; charset=UTF-8」`のように、文字コード(charset)を指定できます。この指定を省略した場合、ブラウザは、文字コードを独自の方法で推定して、推定した文字コードにしたがって画面表示を処理します。たとえば、一部のブラウザにおいては、HTMLテキストの冒頭部分等に特定の文字列が含まれていると、必ず特定の文字コードとして処理されるという挙動が知られています。
> 
> Content-Typeフィールドで文字コードの指定を省略した場合、攻撃者が、この挙動を悪用して、故意に特定の文字コードをブラウザに選択させるような文字列を埋め込んだ上、その文字コードで解釈した場合にスクリプトのタグとなるような文字列を埋め込む可能性があります。
> 
> たとえば、具体的な例として、HTMLテキストに、` 「+ADw-script+AD4-alert(+ACI-test+ACI-)+ADsAPA-/script+AD4-」`という文字列が埋め込まれた場合が考えられます。この場合、一部のブラウザはこれを「UTF-7」の文字コードでエンコードされた文字列として識別します。これがUTF-7として画面に表示されると以下のように扱われるため、スクリプトが実行されてしまいます。
> ```
> <script>alert('test');</script>
> ```
> 
> ウェブアプリケーションが、ウェブページに出力する全ての要素に対して「エスケープ処理」を施し正しくクロスサイト・スクリプティングの脆弱性への対策をしている場合であっても、本来対象とする文字がUTF-8やEUC-JP、Shift_JIS等の文字コードで扱われてしまうと、「+ADw-」等の文字列が「エスケープ処理」されることはありません。
> この問題への対策案として、「エスケープ処理」の際にUTF-7での処理も施すという方法が考えられますが、UTF-7のみを想定すれば万全とは言い切れません。またこの方法では、UTF-7を前提に「エスケープ処理」した結果、正当な文字列（たとえば「+ADw-」という文字列）が別の文字列になるという、本来の機能に支障をきたすという不具合が生じます。
> したがって、この問題の解決策としては、Content-Typeの出力時にcharsetを省略することなく、必ず指定することが有効です。ウェブアプリケーションがHTML出力時に想定している文字コードを、Content-Typeのcharsetに必ず指定してください。

ただし、EC-CUBE4ではSymfony標準でcharset指定されるため問題ありません。
またカスタマイズ時は、コントローラから独自にResponseを返す場合は留意してください。

#### [保険的対策] Cookie情報の漏えい対策として、発行するCookieにHttpOnly属性を加え、TRACEメソッドを無効化する

> 「HttpOnly」は、Cookieに設定できる属性のひとつで、これが設定されたCookieは、HTMLテキスト内のスクリプトからのアクセスが禁止されます。これにより、ウェブサイトにクロスサイト・スクリプティングの脆弱性が存在する場合であっても、その脆弱性によってCookieを盗まれるという事態を防止できます。
> 具体的には、Cookieを発行する際に、「Set-Cookie:(中略)HttpOnly」として設定します。なお、この対策を採用する場合には、いくつかの注意が必要です。
> HttpOnly属性は、ブラウザによって対応状況に差があるため、全てのウェブサイト閲覧者に有効な対策ではありません。
> 本対策は、クロスサイト・スクリプティングの脆弱性のすべての脅威をなくすものではなく、Cookie漏えい以外の脅威は依然として残るものであること、また、利用者のブラウザによっては、この対策が有効に働かない場合があることを理解した上で、対策の実施を検討してください。

EC-CUBE4ではHttpOnly属性は標準で指定済であるため、問題ありません。  
ただし、カスタマイズ時は独自にCookieを発行する場合はHttpOnly属性を指定してください。

※現状ではほとんどのブラウザでHttpOnly属性が活用できます

TRACEメソッドについて：[https://blog.tokumaru.org/2013/01/TRACE-method-is-not-so-dangerous-in-fact.html](https://blog.tokumaru.org/2013/01/TRACE-method-is-not-so-dangerous-in-fact.html)

#### [保険的対策] クロスサイト・スクリプティングの潜在的な脆弱性対策として有効なブラウザの機能を有効にするレスポンスヘッダを返す

> ブラウザには、クロスサイト・スクリプティング攻撃のブロックを試みる機能を備えたものがあります。しかし、ユーザの設定によっては無効になってしまっている場合があるため、サーバ側から明示的に有効にするレスポンスヘッダを返すことで、ウェブアプリケーションにクロスサイト・スクリプティング脆弱性があった場合にも悪用を避けることができます。ただし、下記に示すレスポンスヘッダは、いずれもブラウザによって対応状況に差があるため、全てのウェブサイト閲覧者に有効な対策ではありません。
> 
> 「X-XSS-Protection」は、ブラウザの「XSSフィルタ」の設定を有効にするパラメータです。ブラウザで明示的に無効になっている場合でも、このパラメータを受信することで有効になります。 HTTP レスポンスヘッダに`「X-XSS-Protection: 1; mode=block」`のように設定することで、クロスサイト・スクリプティング攻撃のブロックを試みる機能が有効になります。

ただし、EC-CUBEでは、.htaccessにて以下を指定することにより対策を行っており問題ありません。
```
x-content-type-options: nosniff
x-xss-protection: 1; mode=block
```
※Content Security Policyは未設定

なお、将来的には、X-XSS-Protectionの利用が少なくなり、Content-Security-Policyの利用がスタンダードになると想定されます。

[https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/X-XSS-Protection](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/X-XSS-Protection)

#### [保険的対策] innerHTMLやdocument.writeなど、外部からHTMLタグを注入できるような機能の利用は避ける
DOM XSSについて、EC-CUBEはSPAではないため頻繁にJavaScriptの操作はありませんが、以下のような基本的な対策もカスタマイズ時意識してください。

innerHTMLやdocument.writeなどの、DOM操作の際に外部から指定されたHTMLタグ等が有効になってしまう機能の利用を避け、DOM操作によりテキスト要素を追加するか、textContentプロパティを使用してください。

参考（DomBasedXSSの対策）：「体系的に学ぶ 安全なWebアプリケーションの作り方 第2版」（徳丸浩 著）P.441


## CSRF（クロスサイト・リクエスト・フォージェリ）

### CSRF（クロスサイト・リクエスト・フォージェリ）

#### [根本的解決] 処理を実行するページをPOSTメソッドでアクセスするようにし、その「hiddenパラメータ」に秘密情報が挿入されるよう、前のページを自動生成して、実行ページではその値が正しい場合のみ処理を実行する
> 具体的な例として、「入力画面 → 確認画面 → 登録処理」のようなページ遷移を取り上げて説明します。まず、利用者の入力内容を確認画面として出力する際、合わせて秘密情報を「hidden パラメータ」に出力するようにします。この秘密情報は、セッション管理に使用しているセッションIDを用いる方法の他、セッションIDとは別のもうひとつのID（第2セッションID）をログイン時に生成して用いる方法等が考えられます。生成するIDは暗号論的擬似乱数生成器を用いて、第三者に予測困難なように生成する必要があります。  
> 
> 次に確認画面から登録処理のリクエストを受けた際は、リクエスト内容に含まれる「hiddenパラメータ」の値と、秘密情報とを比較し、一致しない場合は登録処理を行わないようにします。このような実装であれば、攻撃者が「hiddenパラメータ」に出力された秘密情報を入手できなければ、攻撃は成立しません。  
> なお、このリクエストは、POSTメソッドで行うようにします。これは、GET メソッドで行った場合、外部サイトに送信されるRefererに秘密情報が含まれてしまうためです。

EC-CUBEでは、POSTメソッドでのtokenチェックによる実装を行っており、問題ありません。

##### 1. フォーム入力

```php
$form->isSubmitted() && $form->isValid()
```

##### 2. ajax利用時
ajax送信時は、常にcsrf トークンが送信される
```php
$request->isXmlHttpRequest() && $this->isTokenValid()でのチェックを行う
```

##### 3. token-for-anchor

削除ボタンなど、リンク押下での更新処理を行う場合は、token-for-anchorを利用する

```
 csrf_token_for_anchor() 
```


カスタマイズ時の注意点としては、EC-CUBE本体での実装を参考に、トークンチェックを行ってください。  
特に、ajaxやリンクでの更新処理が漏れやすいため注意が必要です。

#### [根本的解決] 処理を実行する直前のページで再度パスワードの入力を求め、実行ページでは、再度入力されたパスワードが正しい場合のみ処理を実行する
> 処理の実行前にパスワード認証を行うことにより、CSRFの脆弱性を解消できます。
> 
> ただし、この方法は画面設計の仕様変更を要する対策であるため、画面設計の仕様変更をせず、実装の変更だけで対策をする必要がある場合には、「処理を実行するページをPOSTメソッドでアクセスするようにし、その[hiddenパラメータ]に秘密情報が挿入されるよう、前のページを自動生成して、実行ページではその値が正しい場合のみ処理を実行する。」「Refererが正しいリンク元かを確認し、正しい場合のみ処理を実行する。」の対策を検討してください。
> 
> この対策方法は、「処理を実行するページをPOSTメソッドでアクセスするようにし、その[hiddenパラメータ]に秘密情報が挿入されるよう、前のページを自動生成して、実行ページではその値が正しい場合のみ処理を実行する。」と比べて実装が簡単となる場合があります。たとえば、セッション管理の仕組みを使用しないでBasic認証を用いている場合、対策をするには新たに秘密情報を作る必要があります。このとき、暗号論的擬似乱数生成器を簡単には用意できないならば、この対策の方が採用しやすいと言えます。

またカスタマイズ時、**remember meを利用しており重要情報の更新の際は、再ログインを促すよう実装してください。**

※再ログイン（パスワードの再入力）は、重要な処理の”直前”であることに留意してください。3画面に渡る処理やウィザード形式での処理において、途中のページでパスワードを入力させた場合、実装によってはCSRF脆弱性が混入する可能性があります。

#### [根本的解決] Refererが正しいリンク元かを確認し、正しい場合のみ処理を実行する

> Refererを確認することにより、本来の画面遷移を経ているかどうかを判断できます。
> 
> Refererが確認できない場合は、処理を実行しないようにします。またRefererが空の場合も、処理を実行しないようにします。これは、Refererを空にしてページを遷移する方法が存在し、攻撃者がその方法を利用して CSRF攻撃を行う可能性があるためです。
> 
> ただし、ウェブサイトによっては、攻撃者がそのウェブサイト上に罠を設置することができる場合があり、このようなサイトでは、この対策法が有効に機能しない場合があります。また、この対策法を採用すると、ブラウザやパーソナルファイアウォール等の設定でRefererを送信しないようにしている利用者が、そのサイトを利用できなくなる不都合が生じる可能性があります。本対策の採用には、これらの点にも注意してください。

EC-CUBEではRefrerを確認する処理はしておりません。ただし、tokenを発行することでリクエスト元が正しいかのチェックを行っております。

カスタマイズ時は、tokenチェックの処理を欠かさないようお願いたします。
上述の「処理を実行するページをPOSTメソッドでアクセスするようにし、その「hiddenパラメータ」に秘密情報が挿入されるよう、前のページを自動生成して、実行ページではその値が正しい場合のみ処理を実行する」参照

#### [保険的対策] 重要な操作を行った際に、その旨を登録済みのメールアドレスに自動送信する
> メールの通知は事後処理であるため、CSRF 攻撃を防ぐことはできません。しかしながら、実際に攻撃があった場合に、利用者が異変に気付くきっかけを作ることができます。なお、メール本文には、プライバシーに関わる重要な情報を入れない等の注意が必要です。

EC-CUBEでは以下のプルリクエストで実装の検討が進んでおります。
[https://github.com/EC-CUBE/ec-cube/pull/5886](https://github.com/EC-CUBE/ec-cube/pull/5886)
カスタマイズ時はこちらを参考にメール通知の実装をお願いたします。

#### [保険的対策] Cookie（セッションID）のSameSite属性には、Laxを指定する。
Cookie（セッションID）のSameSite属性について、ECサイトには決済関連の通信が存在するため、CSRF対策の一つとしてSameSite属性について、決済モジュール等からの指示がない限り、Cookie（セッションID）のSameSite属性にはLaxを指定する。

```
Set-Cookie: flavor=choco; SameSite=Lax
```

[](## HTTPヘッダ・インジェクション)
[](## メールヘッダ・インジェクション)
[](## クリックジャッキング)
[](## バッファオーバーフロー)
## アクセス制御や認可制御の欠落

### アクセス制御の欠落

#### [根本的解決] アクセス制御機能による防御措置が必要とされるウェブサイトには、パスワード等の秘密情報の入力を必要とする認証機能を設ける

> ウェブサイトで非公開とされるべき情報を取り扱う場合や、利用者本人にのみデータの変更や編集を許可することを想定する場合等には、アクセス制御機能の実装が必要です。
> しかし、たとえば個人情報を閲覧する機能にアクセスするにあたり、メールアドレスのみでログインできてしまうウェブサイトが、脆弱なウェブサイトとして届出を受けた例があります。一般に、メールアドレスは他人にも知られ得る情報であり、そのような情報の入力だけで個人情報を閲覧できてしまうのは、アクセス制御機能が欠落していると言えます。  
> パスワード等（みだりに第三者に知らせてはならないものとして一般に考えられている情報）の入力を必要とするようにウェブアプリケーションを設計し、実装してください。

ただし、EC-CUBE4系ではSymfonyのsecurity/firewallを利用しており、管理画面およびマイページは標準でID・パスワードでの認証がかかるため問題ありません。
またカスタマイズ時は、security/firewallの正しい扱い方管理画面の認証制御について注意してください。

### 認可制御の欠落

#### [根本的解決] 認証機能に加えて認可制御の処理を実装し、ログイン中の利用者が他人になりすましてアクセスできないようにする
> ウェブサイトにアクセス制御機能を実装して、利用者本人にのみデータの閲覧や変更等の操作を許可する際、複数の利用者の存在を想定する場合には、どの利用者にどの操作を許可するかを制御する、認可(Authorization)制御の実装が必要となる場合があります。アクセス制御機能が装備されたウェブアプリケーションの典型的な実装では、ログインした利用者にセッションIDを発行してセッション管理を行い、アクセスごとにセッションIDからセッション変数等を介して利用者IDを取得できるように構成されています。
> 
> 単純な機能のウェブアプリケーションであれば、その利用者IDをキーとしてデータベースの検索や変更を行うように実装することができ、この場合は、利用者のデータベースエントリしか操作されることはないので、認可制御は結果的に実装されていると言えます。
> 
> しかし、ウェブサイトによっては、利用者IDをURLやPOSTのパラメータに埋め込んでいる画面が存在することがあります。そのような外部から与えられる利用者IDをキーにしてデータベースを操作する実装になっていると、ログイン中の利用者ならば他の利用者になりすまして操作できてしまうという脆弱性となります。
> 
> これは、認可制御が実装されていないために生じる脆弱性です。データベースを検索するための利用者IDが、ログイン中の利用者IDと一致しているかを常に確認するよう実装するか、または、利用者IDを、外部から与えられるパラメータから取得しないで、セッション変数から取得するようにします。また、他の例として、たとえば注文番号等をキーとしてデータベースの検索や変更を行う機能を持つウェブアプリケーションの場合、注文番号がURLやPOSTのパラメータで与えられる実装になっていると、ログイン中の利用者であれば、他人用に発行された注文番号をURLやPOSTのパラメータに指定することによって、他の利用者にしか閲覧できないはずの注文情報等を閲覧することができてしまう脆弱性が生じることがあります。
> 
> これも、認可制御が実装されていないために生じる脆弱性です。データベースを検索するための注文番号が、ログイン中の利用者に閲覧を許可された番号であるかどうかを常に確認するように実装してください。

なお、EC-CUBE4系では以下を行っております。

* マイページについて
    * ログインセッションから対象のデータを抽出し、表示・更新を実施
* 管理画面
    * 権限管理にて、ページごとの閲覧権限が付与

またカスタマイズ時は、フロント受注やお届け先など、会員と紐づく情報を取得する場合は、`id`だけでなく、ログインセッションから取得した`customer_id`を検索条件に含めてください。
