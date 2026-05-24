---
title: Reactで「取り消し機能」を実装しようとして気づいた設計ミス
emoji: ⚾
type: tech
topics: [React, JavaScript]
published: false
---

## はじめに

草野球の投球記録アプリを個人開発している中で、undo（取り消し）機能を実装しようとしたとき、それまでの設計に根本的な問題があることに気づきました。

この記事では、その問題と解決策、そして背景にある設計原則を記録します。

## 何が起きたか

投球記録画面では以下の state を管理していました。

```js
const [pitches, setPitches] = useState([])       // 投球履歴
const [balls, setBalls] = useState(0)             // ボール数
const [strikes, setStrikes] = useState(0)         // ストライク数
const [outs, setOuts] = useState(0)               // アウト数
const [batterNumber, setBatterNumber] = useState(1) // 打者番号
```

ゾーンをタップするたびに `recordPitch()` を呼びつつ、各 state を手動で更新していました。

undo を実装しようとしたとき、問題が発覚します。

> 直前の球を取り消したい。でも balls・strikes をどうやって1球前に戻す？
>
> pitches 配列には `{ zone, result, pitchType }` しか入っていない。カウントの情報がない。

## 最初に思いついた解決策（案A）

pitches の各要素に、その球を投げる前のカウントも持たせる。

```js
setPitches([...pitches, {
  zone, result, pitchType,
  balls, strikes, batterNumber  // ← 追加
}])
```

undo 時はこれを参照してカウントを復元する。

**案Aのデメリット**

- `setPitches` を呼ぶ箇所が複数あり、すべてに追加が必要
- 1箇所でも漏れるとカウントが壊れる
- データの重複が生じる

## 根本的な問題に気づく

「なぜこんなに複雑になるのか」を考えたとき、本質的な問題が見えてきました。

`balls`・`strikes`・`outs`・`batterNumber` は `pitches` 配列から計算できる値なのに、独立した state として持っていた。

これは React の設計原則の違反です。

> 「他の state から計算できる値は state にしない」
> — React 公式ドキュメント

## Single Source of Truth に基づく設計

`pitches` 配列だけを state として持ち、カウントはそこから毎回導出します。

```js
const [pitches, setPitches] = useState([])

// カウントは導出
const { balls, strikes, outs, batterNumber } = deriveGameState(pitches)
```

`deriveGameState` は pitches 配列を先頭から処理して現在のカウントを返す純粋関数です。

**undo がシンプルになる**

```js
const handleUndo = () => {
  deletePitch(pitches.at(-1).id)   // DBから削除
  setPitches(pitches.slice(0, -1)) // stateから削除
}
```

これだけで `balls`・`strikes` も自動的に正しい値に戻ります。同期ミスが起きようがない。

## 教訓

**1. 派生値を独立した state にしない**

他の state から計算できる値は state にしてはいけません。元データを更新するたびに派生値も手動で同期する責任が生まれ、バグの温床になります。

**2. undo の難しさは設計ミスのサイン**

「取り消し機能が複雑になる」と感じたら、それは state 設計が Single Source of Truth になっていないサインかもしれません。適切に設計されていれば、undo は「配列の末尾を削除するだけ」になるはずです。

**3. Single Source of Truth**

同じ情報が複数の場所に存在すると、それらが矛盾する可能性が生まれます。ソフトウェア設計全般に通じる原則で、「信頼できる唯一の情報源を持つ」ことが整合性を保つ鍵です。

## まとめ

|  | 修正前の設計 | 修正後の設計 |
|---|---|---|
| state の数 | 多い（派生値も持つ） | 少ない（元データのみ） |
| undo の実装 | 複雑 | 配列末尾を削除するだけ |
| 整合性の保証 | 手動で同期が必要 | 自動（導出するため） |
| 拡張性 | 低い | 高い |

機能を追加しようとしたときに「なぜか複雑になる」と感じたら、設計を疑ういい機会です。
