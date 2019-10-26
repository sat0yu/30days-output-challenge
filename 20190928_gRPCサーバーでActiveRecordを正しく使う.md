先日、データベースのマスタスレーブを入れ替えたところ、Ruby実装のgRPCサーバーで大量のエラーが発生する事案が起きた。
gRPCサーバーはActiveRecordを用いてデータベースへクエリを発行しているが、ログにはエラーメッセージ `ActiveRecord::StatementInvalid: PG::UnableToSend: SSL SYSCALL error: EOF detected` が大量に発生しており、データ取得箇所で例外を送出していることが一目瞭然だった。
マスタスレーブを変更しただけでDNSなどは変更しておらず、したがって一度切断された接続でも再接続すれば問題なく利用できるはずである。

本日は上記の体験を通して学んだgRPCサーバーでActiveRecordを使う場合に必ず必要となる処理と、それを実現するInterceptorについて述べる。

まず、RubyのgRPC実装がどのようにしてクライアントリクエストを処置しているのか確認する。
gRPCサーバーはプロセス起動時に予め設定された個数のスレッドをプールし、以降受信した各リクエストはいずれかのスレッドに割り当てられて処理される。
処理を完了したスレッドは破棄されることなくそのままプール（キュー）に戻り、次の呼び出しを待機する。
https://github.com/grpc/grpc/blob/master/src/ruby/lib/grpc/generic/rpc_server.rb#L182-L191

つぎにActiveRecordの挙動もおさえておく。
ActiveRecordはクエリを実行するたびにデータベースとのコネクションを張り直すのではなく、指定された数のコネクションをプールして使い回す。
ただし各Threadがそれぞれコネクションのキャッシュを行っているため、DBコネクションプールの数がサーバーのThread数を上回っていてもあまり意味がないことに注意されたい。
https://github.com/rails/rails/blob/5-2-stable/activerecord/lib/active_record/connection_adapters/abstract/connection_pool.rb#L376-L383

逆に言えば、一度ThreadでキャッシュされてしまったコネクションはThreadが生き続ける間使い回されることを意味する。
では、筆者が経験したように、何らかの要因によって一旦コネクションが破壊され、その後に接続可能に復帰した場合はどうなるだろうか。

ここでRuby on RailsとgRPCの差が見られる。
Ruby on Railsでは何事もなくコネクションが再構築される一方、素朴にgRPCでActiveRecordを使った場合はコネクションの再構築は行ってくれない。
より正確に言えば、コネクションが利用できな状況に陥ったことを認識すらできない。
なぜなら、Threadにキャッシュされたコネクションは自らが不正な状況にあることを知る機構を備えていないからだ。

Railsはリクエストを受信するたびにDBコネクションプールからコネクションをチェックアウト（取り出し）し、リクエスト処理が終わるとチェックイン（返却）する。
そしてチェックアウト時には取り出すコネクションが本当に有効なものかどうか検証しているのだ。
これによって、一度不正状態に陥ったコネクションが再構築されるため、Railsの場合には問題なく動作しつづけたのだ。

一方、gRPCではそのような便利な機能は提供されていない。そして、無いのなら自ら作る他ない。
幸い、gRPCが提供しているInterceptorという機構を用いて簡単に解決できた。
Interceptorの説明は本文の目的からは外れるため、他の記事を参照されたい。

ここで重要なのは、Interceptorという機構の上で何を行えばいいかである。
じつはその答えは、オフィシャルドキュメントにしっかりと書いてあった。

https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/ConnectionPool.html
> Use ActiveRecord::Base.connection_pool.with_connection(&block), which obtains a connection, yields it as the sole argument to the block, and returns it to the pool after the block completes.

すなわち、リクエスト処理のコードを`ActiveRecord::Base.connection_pool.with_connection`で囲むだけで済む。
これによってRails同様にしてリクエストの前後でコネクションをチェックアウト・インし、さらに有効性も検証できる。
以下に、Interceptorとして実装した具体的なコードを添付しておく。
https://gist.github.com/sat0yu/22676846bf8b6125afadb08f956844cf
