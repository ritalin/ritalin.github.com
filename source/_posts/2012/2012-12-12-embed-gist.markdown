---
layout: post
title: "gist.github.com埋め込みテスト"
date: 2012-12-12 21:14
comments: true
categories: 
---

[gist](gist.github.com)のリニューアルで、埋め込んだコードが崩れだしたので確認。

{% gist 4147464 RhinoMocksCreationExtensions.cs %}

相変わらず、行番号はずれたままだけど、コードはそれっぽく表示されるようにはなった。

Octopressでの変更点は、sass/parts/_article.scssの以下の行を削除した

* line 24 ( line-height: 2 )
* line 25 ( text-align: justify ) 
* line 98 - 100 ( td{text-align: center;} )

一桁の行番号が、横並びになるのは、github.comが発行するgistのjavascriptに書かれたhtmlの不備っぽい。
具体的には、

* td class="line_numbers"を
* td class="line_numbers gist-line-numbers"

にしたら、治ったように見えた（がんばって報告した）。

やれやれだぜ。