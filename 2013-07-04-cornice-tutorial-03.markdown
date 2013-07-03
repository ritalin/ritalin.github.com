	---
layout: post
title: "Cornice Tutorial (3)"
date: 2013-06-12 21:50
comments: true
categories: python cornice
---

## はじめに
公式サイトの[チュートリアル](http://cornice.readthedocs.org/en/latest/tutorial.html)の写経、第三回。
ユーザー管理サービスの実装において、Validatorの話がでてくるので、内容を抑えつつ一覧、追加、削除の実装を進めていきたいと思います。

<!-- more -->

## サービスの処理フロー
サービスの各処理はその前後に処理を挟むことができます。前方に挟む方をValidator、後方をFilterと呼びます。
処理の指定はサービス定義、またはデコレータをHTTPメソッドのアノテーションとして使用し、その引数にその関数を渡すことで行います。

即ち処理フローは、

* Validator（オプション）
* 本処理（必須）
* Filter（オプション）

となります。

## Validator
Validatorとは、サービス実装コードの本処理の全面で実行されるプリプロセスであり、CorniceにおけるValidatorの責務は

* リクエスト入力値のチェック
* リクエスト入力値の変換（例えば任意のオブジェクトへ等）

とされています。
ここで入力値として、クエリパラメータやPOSTパラメータだけでなく、HTTPヘッダも対象とすることができます。

Validatorの指定はサービスの定義またはメソッドアノテーションの引数として以下のように行います。

> (サービス定義)

> service = Service(..., validators=<関数>, ...)

> （メソッドアノテーション）

> @service.Get(..., validators=<関数>, ...)

Validator関数のインターフェースは、Requestオブジェクトを受け取り、何も返さない手続きです。

入力値が妥当な場合、Request#validatedディクショナリに、エラーの場合、Request#errorsディクショナリにそれぞれ値を格納します。
実例は後の写経の節で。

Requestは基盤として使用しているPyramidのものをベースに、Validate結果格納用のディクショナリが付与されたものとなっています。
Requestオブジェクトを介して受け取れる値については[APIリファレンス](http://docs.pylonsproject.jp/projects/pyramid-doc-ja/en/latest/api/request.html#pyramid.request.Request)を確認してください。

Request#errorsディクショナリに１つでも値が格納された場合、本処理は実行されず、組み込みのエラーハンドラが実行されます。

## エラーハンドラ
組み込みのエラーハンドラは、正常処理の時と同様に、simpleJsonを使用します。

このエラーハンドラもまたサービスの定義またはメソッドアノテーションの引数として指定することができます。

> (サービス定義)

> service = Service(..., error_handler=<関数>, ...)

> （メソッドアノテーション）

> @service.Get(..., error_handler=<関数>, ...)

エラーハンドラ関数のインターフェースは、エラー内容を引数に取り、[webob.exc.HTTPError](http://pylons-webframework.readthedocs.org/en/v0.9.7/thirdparty/webob.html#module-webob.exc)の派生クラスを返す関数です。

> error_handler_func(errors): Response

エラー内容として、Request#errorsディクショナリが渡されます。

どのように実装するかは、corniceの`util.py`に記述されている、`json_error`関数を眺めてみるのがよいかと思います。

## Filter
Filterは、サービス実装コードの本処理から返されるレスポンスを加工する目的のために使用されます。

Filterの指定はValidator同様に、サービス定義またはメソッドアノテーションの引数として以下のように指定します。

> (サービス定義)

> service = Service(..., filters=<関数>, ...)

> （メソッドアノテーション）

> @service.Get(..., filters=<関数>, ...)

Filterのインターフェースは、レスポンスデータを受け取り、加工されたレスポンスを返す関数です。

> filter_interface(レスポンス): レスポンス

もしくは、オプションとしてRequestオブジェクトを第二引数で受け取ることもできます。

> filter_interface(レスポンス, Request): レスポンス

Filter処理は、corniceの土台であるpyramidのレイヤーで実行されます。
この処理がエラーのエスポンスにも作用するかどうかは、確認できておらず不明です。

## チュートリアルのユーザー管理機能（前回の続き）
前回の続きとして、ユーザー登録を写経します。

### ユーザー登録
ユーザー登録の仕様は、POSTデータとして、ユーザー名を送信します。

ユニークチェックを行い、重複がなければ、

> ユーザー名: ランダムなハッシュ

をディクショナリに保持させます。

まず、この入力値に対するValidatorを用意します。

```
import os
import binascii

def validate_unique(request):
    name = request.params.get('user', '')

    if name == '':
        request.errors.add('url', 'name', 'user is not passed ...')
    if name in _USERS:
        request.errors.add('url', 'name', 'This user exists !')
    else:
        user = { 'name':name, 'token': binascii.b2a_hex(os.urandom(20)) }
        request.validated['user'] = user
```

このValidatorを指定したユーザー登録処理を実装します。

```
@users.post(validators=validate_unique)
def create_user(request):
	"""Add a new user"""
	user = request.validated['user']

	_Users[user['name']] = user['token']

	return { 'token': '%s-%s' % (user['name'], user['token']) }
```

`pserve cornice_tutorial.ini` でサービスを起動。

> curl -X POST http://localhost:6543/users -d user=ktz

の場合

> {"token": "ktz-b'04afcabc1348c9253be856ee81cde78a75185ca9'"}

という出力が返され、
再度同じリクエストを送ると、

> {"status": "error", "errors": [{"location": "url", "description": "This user exists !", "name": "name"}]}

というエラー出力が、

> curl -X POST http://localhost:6543/users

の場合、

> {"errors": [{"description": "user is not passed ...", "name": "name", "location": "url"}], "status": "error"}

が返され、Validatorが作用しているのが分かります。

また、

> curl http://localhost:6543/users

のリクエストで、

> {"users": {"ktz": "04afcabc1348c9253be856ee81cde78a75185ca9"}}

と返され、問題なく追加されいるようです。

### 補足・・・

> curl -X POST http://localhost:6543/users -d user=りたりん

の場合、レスポンスは

> {"token": "\u308a\u305f\u308a\u3093-b'a1ef940c4259833e8bd21da6b1a32a9b8b7e788f'"}

のように、日本語が文字参照に変換された形となります。
これは、規定のレンダラである`json_renderer`が行っています。

### つづく








