Shadow Proxyとは、ある環境（ほとんどの場合はProduction）に送られるトラフィックを複製し他の環境へと送る技術をいう。
nginx 1.13.4からngx_http_mirror_moduleなるモジュールが追加され、特別なプラグインに頼らずともshadow proxyとしての機能を利用できるようになった。
ref. http://nginx.org/en/docs/http/ngx_http_mirror_module.html

ngx_http_mirror_moduleについては@tagomoris氏が具体例を交えて仔細に説明を与えているので、興味がある方はぜひ参照されたい。
ref. https://tagomoris.hatenablog.com/entry/2018/04/03/102831

本文ではShadow Proxyを調べている間にみつけたライブラリを紹介したい。
[goreplay](https://github.com/buger/goreplay)はShadow Proxyの機能を含んだライブラリで、トラフィックを複製することはもちろん、ファイルに保存して必要なときに復元（再生）することが可能だ。
UNIXの原則に従っており、複数の処理をパイプつなぐような感覚で利用できる。

nginxのngx_http_mirror_moduleモジュールの場合には複製元のトラフィックすべてが複製処理を通過する必要があった。
万が一設定が間違っていれば本来のトラフィックを邪魔してしまったり、あるいは複製処理が計算リソースの逼迫の原因になる可能性があったのだ。

これに対しgoreplayはパケットスニファライブラリ[libpcap](https://en.wikipedia.org/wiki/Pcap)を用いて動作する。
すなわち、同一ホストのネットワークインターフェースを監視し通過するトラフィックを盗み見るようにして動作する。
本アプローチならば、アプリケーションの既存インフラには手を加えずとも、goreplayをサーバーと同一ホストで動かすだけ済む。

githubリポジトリにはサンプルWebサーバーの実装も格納されており、手元のマシンで簡単にgoreplayの動作確認ができる。
ref. https://github.com/buger/goreplay/wiki/Getting-Started
リポジトリをチェックアウトしておき、さらにgoreplayのバイナリをダウンロードすれば下記のコマンドで挙動の確認ができる。

# カレントディレクトリのファイル一覧を公開する簡易サーバーを8001ポートで立ち上げる。
gor file-server :8001
# 別のターミナルでポート番号8002を指定して予約する。
gor file-server :8002
# 8001ポートに入ってきたトラフィックを8002ポートへと転送。
sudo ./gor --input-raw :8001 --output-http="http://localhost:8002”

なお、個人的には、Proバージョンを購入するとバイナリプロトコル（thriftやprotocol-buffers）にも対応するという点が面白いとおもった。
https://github.com/buger/goreplay/wiki/%5BPRO%5D-Replaying-Binary-protocols
