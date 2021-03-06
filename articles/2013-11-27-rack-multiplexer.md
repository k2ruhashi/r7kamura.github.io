---
title: Rack::Multiplexer
---

[Rack::Multiplexer](https://github.com/r7kamura/rack-multiplexer)という、複数のRackを束ねるものをつくった。

## Plack寄せ
この前Perl界隈の人達と鍋を囲む機会があって、
!!1;の話、livedoor BlogのPlack化の話、ISUCONの話、
各社古いアプリ抱えていて辛いね苦しいね頑張ろうね若者に1日で書き換えさせようといった話をして、
結局、何となくこの界隈は全体的に「Plack寄せ」が進んでいるねという話に落ち着いた。

## Rack寄せ
一方Ruby界隈だと比較的皆Rackに寄っている傾向にはあると思うけど、
もっと寄せてみると面白いんじゃないかと思って、Rack::Multiplexerをつくった。既にありそう。
Rack::Multiplexerは、所謂WebアプリのRouter(=Dispatcher)の処理を行うための実装で、
メソッドやパスの規則に従って受け取ったリクエストを別のRack applicationに投げられる。
例えば、GET /x へのリクエストはapp-X、GET /y へのリクエストはapp-Yに、という使い方が出来る。
以下の例は、3つの単純なRackアプリにルーティングを行う様子。


```ruby
multiplexer = Rack::Multiplexer.new
multiplexer.get("/a",  ->(env) { [200, {}, ["a"]] })
multiplexer.post("/b", ->(env) { [200, {}, ["b"]] })
multiplexer.put("/c",  ->(env) { [200, {}, ["c"]] })
multiplexer.call(env)
```

## O(1) Router
折角なので、Rack::Multiplexerのルーティングアルゴリズムでは正規表現による高速化を試みた。
[Router::Boom](https://github.com/tokuhirom/Router-Boom)を参考にした。
素朴な実装のルータでは、パターンと手続きの組を配列に入れておき、
配列の先頭からパターンに一致する手続きを探して実行する。
一方Rack::Multiplexerでは、配列を大きな1つの正規表現に変換しておき、
この正規表現を実行することで一致する手続きを探し当てるという方法を取る。
具体的には、個々のパターンを名前付きの括弧で囲んでキャプチャしながら、
パイプで繋いで1つの正規表現に結合し、一致したキャプチャを調べることで実行すべき手続きを探す
(※Rubyのオブジェクト単位での計算量をO(1)にしたところで、正規表現エンジン内では依然としてO(N)の計算が発生するが、
Ruby側でO(N)で操作するよりは若干速いだろうという考え方)。

```
[/foo/, /bar/, /baz/]
↓
↓変換
↓
/(?<_0>foo)|(?<_1>bar)|(?<_2>baz)/
↓
↓"bar"に適用
↓
#<MatchData "bar" _0:nil _1:"bar" _2:nil>
↓
↓_1のキャプチャが一致
↓
手続きを入れた配列の2番目の要素を実行
```

正規表現のNamed Captureという機能を利用して、個々のパターンに数字付きの名前を割り当てる。
Named Captureを利用したのは、個々のパターンには更にCaptureが含まれる可能性があったので、
それと競合しないようにするため。
これにより、配列を走査せずとも、1つの正規表現を適用するだけで実行すべき手続きを見付けられる。
ちなみに手元の環境で700個程度ルーティングパターンが存在したとき、
この方法により速度が17倍程度高速化することが確認できた。
後日、単純な条件でのパターンの数と速度の関係性を比較してグラフにまとめた。
v0.0.2は配列を走査していた頃の実装で、v0.0.4では正規表現を利用するようになっている。

![](/images/2013-11-27-rack-multiplexer/benchmark.png)

## ネットワーク
コード例ではごく単純なRackアプリを利用したけれど、RailsやSinatraのアプリをそのまま接続することもできる。
RailsのRouterにmountという機能があって、Railsアプリに別のアプリを接続出来るんだけど、
まさにそれと同じようなことがRackの層で実現できる。
複数のアプリをツリー状に繋げられるようになる。
今回はRouterをRack化するようなものを作ったけど、他の部分もRackに寄せていって、
例えばリクエスト前後に行うfilter的な処理とか、validationとか、例外処理とか、
その辺全部個々のRackとその組み合わせによって実現して、
大きなネットワーク構造によって1つのシステムを実現するという風に出来ると面白いと思う。

![](/images/2013-11-27-rack-multiplexer/onion.png)

これまでのRackやPlackに対するイメージと言えば、
Middlewareを入れ子状に組み合わせた上記のようなイメージ([引用元](http://docs.pylonsproject.org/projects/pylons-webframework/en/latest/concepts.html#wsgi-middleware))
だったけど、個々のRackのノードからなるツリー構造のネットワークがあり、
それも実行時に経路が変わったり(例えば特定の条件によるCacheの有無)、
という仕組みがあるともっと面白くなるかもしれない。

## アート活動
Rack::Multiplexerは概念実証として作ったアート作品みたいなもので、
普段の考え方から若干ズレた存在を認識することで、既存の道具の有り様を捉え直せれば良いかなと思ってつくった。
最近つくったSitespecもそうで、人類はとにかくテストを書いているけれど、
テストを書くという行為が一体どういう意味を持っているかについて、
少し考え直してみるためにSitespecが出来た。
テストからWebサイトが出てくるという歪な存在のものをつくることで、人間を混乱させて、
自分がどんな行為を行っていて、そこにどういう意味を持たせようとしているのか、
いま使っている道具は自分にとって一体どういう存在なのか、ということに考えが行くようになれば良いかも、
という考えだった。ここまで書いたことは全部嘘で、本当は何も考えてなくて単純に面白ければいいと思ってつくった。
