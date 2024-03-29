「ある要素aが集合Aに含まれているか否かを判定する」問題（以下、所属判定問題）が与えられたとき、どのような実装を考えるだろうか。
素朴な方法として「集合Aに含まれる要素を走査して判定する」方法が考えられるが、これはO(|A|)時間必要なため効率的だとは言えない。
本文では所属判定問題に対するひとつの解として、確率的データ構造BloomFilterをみていく。

まずはじめに、確率的データ構造とは一体何を指すのだろうか。
確率的データ構造を一言で説明すれば「ある確率で期待しない結果を生じるが、確率的でないデータ構造に比べて圧倒的に効率が良いデータ構造」である。
BloomFilterの場合では「偽陽性の誤検出の可能性をもつ所属判定問題の効率の良い解法」と解釈できる。

BloomFilterの実体は初期値0をもつmビットから成るビット配列Bと、k個のハッシュ関数h_1, …, h_kだ（ビット配列だけを指してBloomFilterとする場合もある。）
ハッシュ関数h_1, …, h_kはすべてビット配列のm個のインデックスを値域としてもつ。
要素aをフィルターBに登録する場合には、k個のハッシュ関数h_iにaを与え、得られた値h_i(a)に相当するビットを1に変更する。
ある要素とハッシュ関数の組み合わせが偶然にも他の組み合わせと同じ位置を出力し、複数の関数によってビット値が変更される場合もある。その場合にはとくに特別な処理は行わず、当該位置のビットを1に変更する。

所属判定問題を解く場合には、予め集合Aのすべての要素をBloomFilterに登録しビット配列Bを得ておく。
判定時には、k個のハッシュ関数に対して順にh_i(a)を計算し、該当する位置のビットが0か1かを確認する。
ひとつでもビットが0となるハッシュ関数が見つかれば、その要素aは本当に集合Aに含まれていない。
逆にすべてのハッシュ関数においてh_i(a)の位置のビットが1であった場合、偽陽性を含んでいるものの、妥当な確率で所属判定問題を正解することができる。

ref.wikiで掲載している図が分かりやすい https://ja.wikipedia.org/wiki/%E3%83%96%E3%83%AB%E3%83%BC%E3%83%A0%E3%83%95%E3%82%A3%E3%83%AB%E3%82%BF

直感的にも理論的にも、フィルタの目が細かい（mが大きい）ほうが偽陽性を抑えることができると分かっている。
あるいは、当たり前ではあるが、登録データ数が増えるほどハッシュ関数の衝突確率も高まるため、偽陽性を高める要因になる。
具体的な偽陽性に関する理論解析はインターネット上でも確認できる。

非常に素朴で簡単なアルゴリズムでありながら、利用例は多岐にわたる。
有名なところではBitcoinのトランザクションフィルタリングでも利用されているようだ。

最後に、golangを用いてBloomFilterを実装したので興味がある方は以下より参照されたい。
https://gist.github.com/sat0yu/01efd33b8c72bd5628aa6913aa9e6be0
