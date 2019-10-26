GraphQLのスキーマ設計において進化可能なデザインを実践するために、GithubのMarc-André氏が記事を残している。
本文はMarc-André氏の記事の日本語訳要約である。

GraphQL Schema Design: Building Evolvable Schemas
https://blog.apollographql.com/graphql-schema-design-building-evolvable-schemas-1501f3c59ed5

GraphQLではAPIクライアント側が必要となるフィールドを明示するため、サーバー側が意図しない（突然フィールドを消すなど）限り、自然と後方互換性を保った開発ができるように設計されている。
また、将来的に削除したいフィールドには “Deprecated” の追加情報を与えることもできる。
そうすることで、スキーマ定義を通してクライアント側に古いフィールドを使い続けないよう促すことができる。

それでもなお、後方互換性を保ちながら開発を進めていくことは難しい。
Marc-André氏の記事では、例を交えながら３つのTipsが提案されている。

# 1. シンプルな構造体よりもオブジェクト型（Prefer Object Types over simpler structures）
以下のCalendarEvent型を題材とする。
CalendarEvent型は「カレンダーイベント（以下、イベント）」を表現するオブジェクトで、イベント名（name）と開始・終了日時を表すDateTime型の配列（timeRange）をもつ。
たとえば、あるイベントの開始日時はtimeRangeフィールドの先頭の要素によって表現される。

```graphql
type CalendarEvent {
  name: String!
  timeRange: [DateTime!]!   # [start_date, end_date]
}
```

もし、イベントが未来と過去のいずれに予定されているか（されていたか）を表現する情報を付け加えたい場合はどうすればよいだろうか。
後方互換性を保ったまま変更を加えるには新しいフィールドを追加するほかない。
なお、フィールド名はAPIクライアントが意図を汲み取りやすいよう”timeRange"を接頭辞にもたせたい。

```graphql
type CalendarEvent {
  name: String!
  timeRange: [DateTime!]!   # [start_date, end_date]
  timeRangeInPasr: Boollean!   # if the time range is in the past
}
```

すでにこの時点でスキーマ定義に危うさが見て取れる。
そもそも開始・終了日時を配列で定義しているため、サーバーとクライアント間で「先頭が開始、末尾が終了」という暗黙的な契約が発生している。
加えて、timeRangeフィールドをそのような形式で定義してしまったため新しい情報を付け加えることが困難であった。

この場合には、専用のオブジェクト型TimeRangeを用意することでより便利な形でスキーマを定義できる。

```graphql
type CalendarEvent {
  name: String!
  timeRange: TimeRange!
}

type TimeRange {
  start: DateTime!
  rnd: DateTime!
  inPast: Boolean
}
```

TimeRange型を定義する方法は、以下の３つの点において優れている。
- (a) TimeRange型に対して互換性維持のコストを考えることなく新しいフィールドを追加可能
- (b) （暗黙的な契約を無くし）すべての情報をフィールドに定義したためクライアントにとって理解が容易
- (c) 接頭辞に頼らずとも関係する情報がひとつのフィールド内で取得可能

フィールドが将来的にどう進化するか想像し、迷いがあるならば専用型の定義を検討されたい。

# 2. 迷ったら特徴的な名前（When in Doubt, Be Specific With Naming）

フィールド名の変更は不可能ではないが、変更を完了するまでにコストがかかる。

例えば、あるブログシステムのAPIを設計しているとして、記事に寄せられた「コメント」を表現するためにCommentという言葉を使っているとしよう。
そのうち「フィードバック」を表現するためにFeedbackFormCommentという名前を使いたい状況が生まれる。
すると、共通の要素を抽象化しCommonインターフェースとしてまとめておきたいと思うだろう。

名前の衝突を解消する場合、以下の手順で変更が行われる。
- (1) 使いたい名前をもつ既存フィールドにDeprecated情報を付加し、一時的な名前で新しいフィールドを作成する
- (2) 使いたい名前をもつ既存フィールドをすべて削除し、一時的な名前で作成したフィールドにDeprecated情報を付加する
- (3) 一時的な名前で作成したフィールドを削除し、使いたかった名前でフィールドを作り直す

本来、記事に寄せられたコメントをPostCommonという名前で表現しておけば、このような多段階のステップを経る変更手続きは必要なかったはずだ。

# 3. カスタムスカラーよりも型（Prefer Fields and Types Over Custom Scalars）

GraphQLが標準で提供する型だけでは表現できない構造がある。
例えば、再帰的な構造をGraphQLで表現するにはカスタムスカラーを用いる必要がある。
しかし、ほとんどのケースではカスタムスカラやJSON型に頼らずとも、GraphQLの標準型を用いることで事足りるはずだ。

カスタムスカラを多用すること関係する問題がいくつかある。
- APIクライアントはカスタムスカラがとる値を知ることが困難
- サーバーサイドはカスタムスカラの値がどのように利用されているか知る術がなく、構造を変更することが困難

たとえば、statusというフィールドがString型を基にしたカスタムスカラーで定義されている場合、クライアントサイドはカスタムスカラーがどのような値をとるのか想像するしかない。
一方でenumを使っていれば、少なくとも可能性のある値のリストを事前に把握することができる。

以上の３点が、Marc-André氏の主張する進化可能なGraphQL Schema設計のテクニックである。
筆者の職場でもGraphQLを利用しているが、ぜひともチーム内で元記事をシェアしたいと思う。

またMarc-André氏は今まさにGraphQLのスキーマ設計に関する書籍を執筆中とのこと。
発売が待ち遠しい。
https://book.graphqlschemadesign.com/
