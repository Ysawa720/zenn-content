---
title: "Expressのリクエストからレスポンスまでを、ソースを読んで追ってみた"
emoji: "🔍"
type: "tech"
topics: ["express", "nodejs", "firebase", "typescript"]
published: true
---

## はじめに

メンターの課題でFirebase製ECサイトを作っている。サインアップのバリデーションミドルウェアを書いていて、ふと「`res.status(400).json(...)` って、結局どこで何が起きてるんだろう」と気になった。

「動くからOK」で流さず、**データがどこに保持され、どのメソッドがトリガーになって、誰に渡されるのか**まで追いたくなり、Express本体とrouterパッケージのソースを読んでみた。その記録。

対象は、以下のようなよくあるミドルウェア。

```ts
export const validateSignupRequest = (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  const errors: string[] = [];
  if (!email) errors.push("メールアドレスは必須です。");
  // ...省略...
  if (errors.length > 0) {
    return res.status(400).json({ message: "入力内容に誤りがあります。", errors });
  }
  return next();
};
```

## まず結論

先に、追いかけた結果わかった全体像を置いておく。

- リクエスト／レスポンスのデータを保持している「箱」は、**終始 Node.js標準の `http.IncomingMessage`（req）と `http.ServerResponse`（res）**。
- Expressが提供するのは「ルーティングテーブルを持つ `app`」と「標準の箱へ書き込む便利メソッド群」。
- ルーティングテーブルの実体は、`app` が直接持つのではなく **`app.router` の `stack` という配列**。
- 通信の節目には `app(req, res)` / `next()` / `res.end()` という3つのトリガーがあり、それぞれ「処理開始」「次への委譲」「最終送信」の合図になっている。

以下、この結論に至るまでを順に見ていく。

## `app` の正体は「関数 + 後付けメソッド」

`const app = express()` の `app` が何なのか。`express()` のソースを見ると、こうなっている。

```js
function createApplication() {
  var app = function(req, res, next) {
    app.handle(req, res, next);
  };
  mixin(app, EventEmitter.prototype, false);
  mixin(app, proto, false); // proto = application.js のメソッド群
  return app;
}
```

`app` の実体は `(req, res, next) => app.handle(...)` という**ただの関数**で、そこに `mixin()` で `.get()` や `.use()` などのメソッドを後付けしている。だから `app(req, res)` と関数として呼べるし、`app.get(...)` とメソッドも生えている、というハイブリッドだった。

そして `app(req, res)` が呼ばれたときの中身は、`app.handle()` に丸投げするだけの薄い窓口にすぎない。

### 寄り道：`mixin()` の2行は何をしているのか

この `mixin()` 2行が、ほぼ空の関数だった `app` を「Expressのapp」に組み立てている。それぞれ役割が違う。

`mixin()` は `merge-descriptors` というライブラリの関数で、シグネチャは `mixin(dest, src, redefine)`。コピー先（`dest`）にコピー元（`src`）のプロパティを混ぜ込むヘルパーで、第3引数 `false` は「既存の同名プロパティがあれば上書きしない」指定。

`Object.assign` と決定的に違うのは、単純な値コピーではなく**プロパティディスクリプタごとコピーする**点。`getOwnPropertyDescriptor` で `get`/`set`/`writable` などの設定を丸ごと取り出して `defineProperty` で定義し直すので、**getter/setter もそのまま移植できる**。`Object.assign` だと getter は評価された「値」になってしまうため、ここが効いてくる（前述の遅延生成getterのような仕組みを壊さずに混ぜられる）。

2行の役割分担はこう。

- `mixin(app, EventEmitter.prototype, false)` → **イベント機能の付与**。`EventEmitter.prototype` はNodeの `events` モジュールのプロトタイプで、`on()`/`emit()`/`once()` などが乗っている。これを混ぜることで `app.on(...)` や `app.emit(...)` が使えるようになり、mountイベントなどをemitできるようになる。
- `mixin(app, proto, false)` → **Express本体の機能の付与**。`proto` は `application.js` 内でローカル定義された、`app.get`/`app.set`/`app.use`/`app.listen` などのExpress独自メソッドをまとめたオブジェクト。

つまり `app` は、最初はほぼ空の関数オブジェクトで、この2回の `mixin` によって「イベントも扱えて、Expressの機能も持つ」ハイブリッドに組み立てられている、という流れだった。

#### `proto` はどこから来るのか：`require` と `mixin` の分業

ここで `proto` の出どころも追っておく。Expressのソースはファイルが分かれていて、`app` の入口と、メソッドの実装が別ファイルになっている。

- `lib/express.js` … `express()` 関数の本体。素の `app` を作り、`mixin` で機能を混ぜて返す**入口**。
- `lib/application.js` … `app.get`/`app.use`/`app.listen` などのメソッドの**実装**をまとめたファイル。前述の `Object.defineProperty(this, 'router', ...)` もここにある。

`express.js` の冒頭で、こう読み込んでいる。

```js
var proto = require('./application');
```

この `require` が `application.js` の `module.exports`（メソッドが一式生えたオブジェクト）を丸ごと受け取り、`proto` に入れている。ここで注意したいのは、`get` だけ・`use` だけと個別に取り出しているのではなく、**詰め合わせを丸ごと1個**受け取っている点。

そして `require` しただけでは `app` はまだメソッドを持たない。`mixin(app, proto, false)` で `app` に**移植して初めて**使えるようになる。役割を分けると、`require` が「材料（`proto`）を持ってくる」係、`mixin` が「材料を `app` に組み付ける」係。`require` はNode.js（CommonJS）の仕組み、`mixin` は `merge-descriptors` の関数で、層の違う2つの道具が組み合わさって「別ファイルに書いたメソッドを `app` に持たせる」を実現している。

ちなみに `application.js` 内のコードで `this` が `app` を指すのも、この `proto` が最終的に `app` に混ぜ込まれ、`app` のメソッドとして実行されるから。だから `Object.defineProperty(this, 'router', ...)` の `this` も、実行時には `app` になる。

## ルーティングテーブルの実体は「配列」だった

次に、`app.use("/products", router)` のような登録が、どこに溜まっているのかを追った。「テーブル」という言葉から連想配列（オブジェクト）を想像していたが、実際は違った。

`app.router` は、`app` が直接データとして持つのではなく、**初回アクセス時に別パッケージ `router` のインスタンスを1回だけ作って返す getter** になっている。

```js
Object.defineProperty(this, 'router', {
  get: function getrouter() {
    if (router === null) {
      router = new Router({ /* ... */ });
    }
    return router;
  }
});
```

### 寄り道：`Object.defineProperty` とは

ここで使われている `Object.defineProperty` も気になったので確認した。シグネチャは `Object.defineProperty(obj, prop, descriptor)` の3引数で、それぞれ「対象オブジェクト」「プロパティ名」「そのプロパティの設定を書いたオブジェクト」を指す。

`obj.x = 1` という普通の代入と違うのは、`descriptor` で挙動を細かく制御できる点。指定できる主なキーは以下。

- `value`: 値
- `writable`: 再代入可能か
- `enumerable`: `for...in` や `Object.keys` で列挙されるか
- `configurable`: 後から削除・再定義できるか
- `get` / `set`: アクセサ（getter/setter）。`value`/`writable` とは併用不可

注意したいのは、**普通の代入で作るプロパティは上記フラグが全部 `true` なのに対し、`defineProperty` は省略すると全部 `false` になる**こと。

#### `app.router` は「値」ではなく「アクセサプロパティ」

ここで一つ勘違いしそうになった。「`app.router` に `{ get: getrouter }` というオブジェクトが格納されている」と読んでしまいそうになるが、そうではない。`descriptor`（`{ get: ... }`）は **`app.router` に入る値ではなく、「`app.router` をどう振る舞わせるか」の設定書**だ。

JavaScriptのプロパティは大きく2種類ある。

- **データプロパティ**（`obj.x = 42`）：値そのものを保持し、読むと値が返る。
- **アクセサプロパティ**（`get`/`set` で定義）：値を保持せず、**読まれた瞬間に `get` 関数が実行され、その戻り値が返る**。

`app.router` は後者。だから `app.router` の中に `get` という構造が残るわけではなく、「読まれたら `getrouter` を走らせる」という**振る舞いだけ**が結びついている。`get` のみで `set` が無いので、外から `app.router = ...` と差し替えられない読み取り専用でもある。

#### 「1回だけ生成」を支えているのはクロージャ

もう一段気になったのが、`getrouter` の外にある `router` 変数の役割。ソースの前後を補うとこうなっている。

```js
var router = null;                     // ← 外側スコープの変数

Object.defineProperty(this, 'router', {
  get: function getrouter() {
    if (router === null) {             // ← 外側の router を参照
      router = new Router({ /* ... */ });
    }
    return router;
  }
});
```

`getrouter` は `router` 変数と**同じスコープで定義されている**ため、生成済みのRouterを覚えておける（＝クロージャ）。初回アクセス時だけ `new Router()` し、2回目以降は `if (router === null)` が `false` になって同じものを返す。この `router` 変数は外部からは触れず、`getrouter` だけが読み書きできる"隠し持ちの記憶場所"として働いている。「外部から隠しつつ、特定の関数だけが触れる変数」は、クロージャがプライベート変数の代わりに使われる典型パターンだった。

まとめると、Expressが `app.router` を `defineProperty` で定義しているのは、`get` を使って**初回アクセス時にRouterを1回だけ生成する**（遅延生成）ためで、その「1回だけ」をクロージャが支えている。単なる代入では実現できない「アクセスされた瞬間に処理を走らせる」挙動を、アクセサプロパティで差し込んでいる、という設計だった。

その `Router` の中身を見ると、`app` と**まったく同じパターン**がもう一段入れ子で使われていた。

```js
function Router(options) {
  function router(req, res, next) {
    router.handle(req, res, next);
  }
  router.stack = []; // ← ここが核心
  return router;
}
```

ルーティングテーブルの実体は、**`router.stack` という配列**。`app.get()` や `app.use()` を呼ぶたびに、

```js
this.stack.push(layer);
```

で末尾に `Layer` が追加されていく。

### なぜオブジェクトではなく配列なのか

Expressのルーティングは「上から順に照合して、最初に一致したものを採用する」という**順序依存**の挙動をする。これは配列（順序を持つ構造）だからこそ自然に実現できる。もし連想配列で持っていたら、「登録した順番」という概念自体が曖昧になってしまう。

「登録順に走査」という挙動の裏付けが、`stack = []` という一行にあった、というのが個人的に一番腑に落ちたポイントだった。

## `res` は「空箱」ではなく「最初から道具が揃ったオブジェクト」

最初、`res` を「レスポンスがこれから格納される空の箱」だと思っていた。だがミドルウェアが受け取った時点で、すでに `.status()` や `.json()` が使える。

正しくは、**最初からレスポンスを組み立てる機能（メソッド）を備えた完成済みのオブジェクト**。「空」なのは道具ではなく、「これから何を送るか（中身）」の方だけだった。リモコンに例えると、ボタンは最初から全部付いていて、まだ押していないだけ、という状態。

## `res.status(400).json(...)` の中で起きていること

ここが本題。1行に見えるが、内部では複数のメソッドが連鎖している。ソースで確認した順に並べる。

### ① `res.status(400)`

```js
res.status = function status(code) {
  // ...バリデーション...
  this.statusCode = code; // ただの代入
  return this;            // 自分自身を返す
};
```

やっているのは `statusCode` への代入だけ。この時点で何かが送信されるわけではなく、「送るときは400番で」という予約にすぎない。`return this` があることで、続けて `.json()` を書けるメソッドチェーンが成立する。

### ② `res.json(obj)`

```js
res.json = function json(obj) {
  var body = stringify(obj, /* ... */);        // JSON文字列化
  if (!this.get('Content-Type')) {
    this.set('Content-Type', 'application/json'); // ヘッダー設定
  }
  return this.send(body);                       // send に委譲
};
```

オブジェクトをJSON文字列に変換し、Content-Typeを設定し、最後は自分では送らず `res.send()` に丸投げする。

### ③ `res.send(文字列)` → `this.end()`

`res.send()` はさらにヘッダー（Content-Lengthなど）を整えたあと、最終的に `this.end(chunk)` を呼ぶ。この `this.end()` こそが、**Express独自ではなくNode.js標準（http.ServerResponse）のメソッド**で、ここで初めて実際のネットワーク送信が起きる。

## `this.end()` が実際に送るもの

`this.end(chunk)` の `chunk` は**本文だけ**。だが実際にネットワークを流れるのは、レスポンス全体だ。

```
HTTP/1.1 400 Bad Request          ← statusCode から Node.js が自動生成
Content-Type: application/json     ← res.json() が設定
Content-Length: 98                 ← res.send() が設定
Access-Control-Allow-Origin: *     ← cors() が設定
（空行）                            ← ヘッダーと本文の区切り
{"message":"...","errors":[...]}   ← chunk（res.json() が JSON.stringify した本文）
```

ステータス行・ヘッダーは `chunk` とは別に、それ以前の `res.status()` や `res.set()`、`cors()` の呼び出しで **`res`（http.ServerResponse）に直接蓄積されていた情報**から組み立てられる。`res.end()` が呼ばれた瞬間、ServerResponse自身がそれらを HTTP の書式に整形して送信する。

ここで最初の結論に戻る。**各Expressメソッドは、Node.js標準の `res` という箱に書き込むためのラッパー**であり、蓄積先は終始 `http.ServerResponse` だった。

## 全体の流れ（POST /signup を例に）

ここまでを1本につなぐ。

```
【初期化】express()でapp生成
        → app.use()を呼ぶたび app.router(遅延生成)の stack配列にLayerをpush
        → Firebase CLIがapi（トリガー）を発見（stackの中身は未解析）
──────────────────────────────────
クライアント
   ↓ HTTPリクエスト
Google/Firebaseインフラ（宛先＝プロジェクトID/リージョン/関数名だけ見る）
   ↓ Node.js環境が req(IncomingMessage)・res(ServerResponse) を生成
   ↓ プレフィックス切り落とし → app(req, res)  ★トリガー1
app（薄い窓口） → app.handle() → app.router に委譲
   ↓ router.stack【配列】を上から順に照合
   ↓ マウント一致ならパスを切り落とし、子routerのstackへ（入れ子）
ミドルウェアチェーン： cors() → validateSignupRequest → controller
   ↓ 各段の next() でリレー（打ち切りも可能）  ★トリガー2
   ↓ 各メソッドが res(ServerResponse) に status・ヘッダー・本文を蓄積
res.end(chunk)  ★トリガー3（Node.js標準）
   ↓ ServerResponseが「蓄積済みstatus＋ヘッダー」＋「chunk」をHTTP書式に整形
実際のネットワーク送信 → クライアント受信
```

`req.url` のプレフィックス切り落とし（Firebase側）と、マウントによるパス切り落とし（Express側）が、**同じ原理の入れ子**になっているのも面白かった。宛先の階層を一段ずつ剥がしながら奥へ渡していく、という構造が繰り返されている。

## `next()` と `return` について補足

ミドルウェアで `return res.status(400).json(...)` と `return` を付けるのは、`res.json()` の戻り値を呼び出し元に返したいからではなく、**この関数の実行をそこで打ち切る**ため。`return` が無いと、エラーがあっても後続の `return next()` まで到達し、本来止めたい処理（登録処理）に進んでしまう。`return` はここでは制御のためだけに使われている。

`next` は、Expressが「stack上の次の要素は何か」を把握した上で、その場で生成して各ミドルウェアに渡している「次へ進むボタン」。ここも `req`/`res` とは出どころが違う点だった。

## おわりに

「1行の `res.status(400).json(...)`」の裏に、`status → json → send → end` という委譲の連鎖と、`http.ServerResponse` への蓄積があった。そして「ルーティングテーブル」の実体が `stack` という配列で、その配列であることがExpressの順序依存の照合を支えていた。

ライブラリの中身は、難しそうに見えても、たどってみると「代入」「配列へのpush」「委譲」といった小さな部品の組み合わせでできている、というのを実感できた題材だった。

（この記事は学習過程の整理です。誤りに気づいた方はコメントで指摘いただけると助かります。）
