---
title: "なぜinterfaceで関数型を書けるのか — call signatureと「呼び出せるオブジェクト」"
emoji: "🔧"
type: "tech"
topics: ["typescript", "javascript"]
published: false
---

## はじめに

TypeScriptの型定義を読んでいると、こんな記述に出会うことがあります。

```typescript
interface EventListener {
  (evt: Event): void;
}
```

`interface` なのにプロパティ名がない。`function` も `=>` もない。最初に見たとき「これは一体どういう構造なんだ？」と引っかかりました。

この記法は **call signature（呼び出しシグネチャ）** と呼ばれるものですが、「なぜこう書けるのか」を腹落ちさせるには、JavaScriptの「関数はオブジェクトの一種である」という事実まで遡る必要があります。

この記事では、`lib.dom.d.ts` の型定義をきっかけに、

- 関数が「呼び出せるオブジェクト」であること
- `interface` で関数型を表現する仕組み
- call signatureが本当に効いてくる場面

を、つまずいた順番のまま整理していきます。

## 出発点：`.d.ts` のメソッド定義

ブラウザの型定義 `lib.dom.d.ts` で `getElementById` の型を見ると、こうなっています。

```typescript
getElementById(elementId: string): HTMLElement | null;
```

`function` も `{}` もありません。これは `.d.ts`（型定義ファイル）が「型の宣言だけ」を書く場所で、実装を持たないからです。さらにこの1行は単独の関数ではなく、`Document` インターフェースのメソッドの一部です。

```typescript
interface Document {
  getElementById(elementId: string): HTMLElement | null;
  querySelector(selectors: string): Element | null;
  // ...
}
```

ここで重要なのは、**interfaceの中にメソッド（呼び出せるプロパティ）を書けている**という点です。これが後の話の土台になります。

## 前提：interfaceはオブジェクトの形を定義する

`interface` はオブジェクトの「形（型）」を定義する機能です。プロパティもメソッドも書けます。

```typescript
interface User {
  name: string;            // プロパティ
  greet(name: string): string;  // メソッド
}
```

メソッドの型の書き方には、実は2通りあります。

```typescript
interface User {
  // ① メソッドシグネチャ記法
  greet(name: string): string;

  // ② プロパティ + 関数型 記法
  greet: (name: string) => string;
}
```

①と②はほぼ同じ意味です。①が `: 戻り値型` を使い、②が `=> 戻り値型` を使う、という構文の違いだけです。

ここで一つ、混乱しやすいポイントを潰しておきます。型を書く場所（interface内）では `=>` の後ろは **戻り値の型** であって、実装の `{}` ではありません。実装の `{}` が登場するのは、実際に値（オブジェクトリテラル）を作るときだけです。

```typescript
// 型の世界（interface内）：=> の後ろは戻り値の型
interface User {
  greet: (name: string) => string;
}

// 値の世界（実装）：=> の後ろが実装ブロック {}
const user = {
  greet: (name: string) => {
    return `Hello ${name}`;
  },
};
```

## 核心：JavaScriptの関数は「オブジェクトの一種」

ここからが本題です。なぜinterfaceで関数の形を書けるのか。答えは **JavaScriptでは関数もオブジェクトの一種だから** です。

その証拠に、関数にはプロパティを足せます。

```javascript
function fn() {
  return "hello";
}

fn.author = "佐藤";   // プロパティを足せる

fn();         // ✅ 関数として呼べる
fn.author;    // ✅ "佐藤"（プロパティにアクセスできる）
```

自分で足さなくても、関数は生成時点でプロパティを持っています。これもオブジェクトである証拠です。

```javascript
function greet(a, b) {}
greet.name;    // "greet"  ← 関数名
greet.length;  // 2        ← 引数の数
```

つまり関数は「**呼び出せて（`fn()`）、かつプロパティも持てる（`fn.prop`）オブジェクト**」= **呼び出せるオブジェクト** です。

仕様レベルで言うと、関数は内部に `[[Call]]`（呼ばれたときの動作）を持った特別なオブジェクトです。オブジェクトとしての性質を備えたうえで、追加で「呼び出せる」能力を持っている、というイメージです。

### 補足：`[[Call]]` とは何か

この `[[Call]]` について、もう少し掘り下げておきます。

まず `[[ ]]` という二重角括弧は、ECMAScript仕様（JavaScriptの公式仕様書）が **内部メソッド / 内部スロット** を表すために使う記法です。「内部」とは、**JavaScriptのコードからは直接アクセスできない**、エンジン内部だけが持つ隠れた仕組み、という意味です。`fn.[[Call]]` のようには書けません。

その上で `[[Call]]` は、**「`( )` を付けて呼び出されたときに何をするか」を定義した内部メソッド** です。オブジェクトがこれを持っているかどうかが、「呼び出せるかどうか」の正体になります。

```javascript
const fn = () => "hello";   // [[Call]] を持つ → 呼べる
const obj = {};             // [[Call]] を持たない → 呼べない

fn();    // ✅ エンジンが fn の [[Call]] を実行する
obj();   // ❌ obj には [[Call]] がない → TypeError
```

普通のオブジェクトと関数の違いを、内部メソッドの観点で並べるとこうなります。

| | 普通のオブジェクト `{}` | 関数 `function(){}` |
|---|---|---|
| プロパティを持てる | ✅ | ✅ |
| `[[Call]]` を持つ | ❌ | ✅ |
| `obj()` で呼べる | ❌ | ✅ |

関数は「普通のオブジェクトが持つ性質（プロパティを持てる）」に加えて、**`[[Call]]` という追加の内部メソッドを持っている**。これが「関数は特別なオブジェクト」と言ったときの中身です。

そして、この `[[Call]]` は**オブジェクトが生成される瞬間にエンジンが設定する**もので、JavaScriptのコードから後付けする手段がありません。後の節で「`{}` で生まれたものに呼べる能力は足せない」と書くのは、まさにこの理由によります。プロパティの追加は「オブジェクトの中身をいじる」操作ですが、`[[Call]]` は「オブジェクトの種類そのもの」に関わるため、外から触れないのです。

## 大事な区別：「オブジェクトが関数を持つ」≠「オブジェクト自体が呼べる」

ここは私が一番つまずいたところです。オブジェクトのプロパティに関数を入れても、**オブジェクト自体が呼べるようにはなりません**。

```javascript
const obj = {
  fn: function () {
    return "hello";
  },
};

obj.fn();   // ✅ プロパティ fn が関数だから呼べる
obj();      // ❌ TypeError: obj is not a function
```

`obj` はただのオブジェクトで、その中の `fn` が関数なだけ。`obj` 自身は `[[Call]]` を持たないので `obj()` とは呼べません。

さらに、「オブジェクトリテラルの直下に名前なしで関数を置く」こともできません。オブジェクトリテラル `{}` の中身は必ず「キー: 値」のペアである必要があるからです。

```javascript
const obj = {
  () => "hello"   // ❌ 構文エラー。キーがない
};
```

### 「呼べるオブジェクト」を作る唯一の方法

`obj()` と呼べる値を作るには、**土台を関数にして、そこにプロパティを足す**しかありません。`{}` から始めてはダメです。

```javascript
function obj() {
  return "呼ばれた";
}
obj.prop = "値";

obj();      // ✅ "呼ばれた"  ← 土台が関数だから呼べる
obj.prop;   // ✅ "値"
```

ポイントは、**「呼べるかどうか」は値が生成された瞬間に決まる** ということです。

```javascript
const a = {};            // オブジェクトとして生成 → 一生 a() はできない
const b = function(){};  // 関数として生成 → b() できる、かつ b.x も足せる
```

先述の [[Call]]に関する補足の通り生まれが違うので、後から行き来できません。「関数にプロパティが付くことはできるが、オブジェクトが関数になることはできない」と覚えておくと整理しやすいです。

## 合流点：call signature

ここまでで2つの事実が揃いました。

1. **interfaceはオブジェクトの形を定義する機能**
2. **関数はオブジェクトの一種（呼び出せるオブジェクト）**

関数がオブジェクトの一種なら、「オブジェクトの形を定義するinterface」で関数の形も表現できるはずです。でも、普通のプロパティ記法 `name: 型` では「**この型自体が呼び出せる**」という性質を書けません。

そこで用意されたのが、**名前なしのシグネチャ = call signature** です。

```typescript
interface EventListener {
  (evt: Event): void;
}
```

メソッド定義と並べると意味がはっきりします。

```typescript
interface X {
  greet(evt: Event): void;   // greet というキーで呼べる → x.greet(e)
  (evt: Event): void;        // キーなし → x 自体を呼べる   → x(e)
}
```

`greet` という名前を外すと「どのキーで呼ぶか」の指定が消え、「キーを介さず、その値を直接呼べる」という意味に変わります。これがcall signatureの正体です。

なお、「名前を取り除ける」のはJavaScriptの実行時仕様が直接そう定めているからではなく、TypeScriptが型を表すために設計した文法です。ただしその設計は、JavaScriptに「呼び出せるオブジェクト」が実在するという土台の上に乗っています。**実在する値の形を、TSが型として写し取るための記法** だと捉えると正確です。

## 関数型を書く3つの方法

実は、単なる関数型を書くだけなら方法は複数あり、どれも同じ型を表します。

```typescript
// ① type で書く（一番素直）
type EventListener = (evt: Event) => void;

// ② interface の call signature で書く
interface EventListener {
  (evt: Event): void;
}

// ③ 変数の型注釈に直接書く
const handler: (evt: Event) => void = (evt) => {};
```

つまり `EventListener` 程度の単純な関数型なら、`type` でも `interface` でも好きな方で書けます。call signatureが**必須**というわけではありません。

## call signatureが本領を発揮する場面

では、call signatureはいつ本当に必要になるのか。それは **「呼び出せて、なおかつプロパティも持つ」型** を表現したいときです。これは `type Fn = () => void` の矢印記法だけでは書けません。

```typescript
interface Counter {
  (start: number): number;   // 呼び出せる ← call signature
  count: number;             // プロパティも持つ
  reset(): void;             // メソッドも持つ
}
```

これはまさに、前半で見た「呼び出せるオブジェクト」を型として表現したものです。設定値・補助メソッドを抱えたライブラリの関数など、「メイン機能は関数呼び出し、付随機能はメソッド」という設計は割とあるみたいです。

```javascript
function api(path) {
  return fetch("/api/" + path);
}
api.baseUrl = "https://...";
api.clearCache = function () { /* ... */ };

api("users");      // 関数として
api.baseUrl;       // プロパティとして
api.clearCache();  // メソッドとして
```

この `api` を型で書こうとすると、call signatureが必要になります。

```typescript
interface Api {
  (path: string): Promise<Response>;  // 呼び出せる
  baseUrl: string;                    // プロパティを持つ
  clearCache(): void;                 // メソッドを持つ
}
```

## まとめ

最初の疑問に戻ります。

```typescript
interface EventListener {
  (evt: Event): void;
}
```

なぜこう書けるのか。流れで言うと、

1. JavaScriptでは関数もオブジェクトの一種で、「呼び出せるオブジェクト」である
2. その「呼び出せる」性質を型で表現する必要がある
3. interfaceはオブジェクトの形を書く機能なので、**名前を外したシグネチャ（call signature）** で「この型自体が呼び出せる」と表現できるようにした

ということでした。

そして、

- 単純な関数型なら `type` でも `interface`（call signature）でも書ける
- call signatureの真価は、「関数であり同時にオブジェクトでもある」値 = **呼び出せるオブジェクト** を型として表現できる点にある

「関数=呼び出せるオブジェクト」と「interface=オブジェクトの形の定義」という2つの理解が合流した地点が、まさにcall signatureだった、というのが今回の結論です。型定義ファイルの一行で生じた疑問を掘り下げることで、根っこの仕様にたどり着くことができました。
