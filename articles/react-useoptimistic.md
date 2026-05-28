---
title: "useOptimistic + useTransition でタップ即反映UIを作った話"
emoji: "⚾"
type: "tech"
topics: ["react", "nextjs", "typescript"]
published: true
---

## はじめに

草野球の投球記録アプリを個人開発しています。記録画面で5×5のストライク/ボールゾーンをタップすると Supabase にデータを保存するのですが、DB通信が終わるまでカウントが変わらず、レスポンスが遅い体験になっていました。

これを解決するために `useOptimistic` と `useTransition` を使って楽観的UIを実装したので、その仕組みをまとめます。

---

## 問題：DB通信が終わるまでUIが変わらない

タップしてからカウントが更新されるまでの流れはこうでした。

```
タップ → recordPitch（DB通信・数百ms） → setPitches → UI更新
```

`setPitches` は「次のレンダリングでこの値を使ってくれ」という予約です。DB通信が終わるまで `setPitches` を呼べないため、通信中はUIが変わりません。

---

## 解決策：useOptimistic

`useOptimistic` は「本物のstateとは別に、仮の楽観的なstateを持つ」仕組みです。

```js
const [optimisticPitches, addOptimisticPitches] = useOptimistic(
  pitches,
  (state, newPitch) => [...state, newPitch]
);
```

**第1引数**：本物のstate（`pitches`）。`optimisticPitches`（=仮のstate） はこの引数を元に作られます。

**第2引数**：更新関数。`addOptimisticPitches` を呼んだときに「`optimisticPitches`をどう変えるか」を定義します。
- `state`：React が自動で注入する現在の `optimisticPitches`
- `newPitch`：`addOptimisticPitches` に渡した引数

これで `pitches`（本物）と `optimisticPitches`（仮）の2つを持てます。

```
pitches（本物）         → DB保存が終わったら setPitches で更新
optimisticPitches（仮） → addOptimisticPitches を呼んだ瞬間に即更新
```

画面の描画には `optimisticPitches` を使うので、DB更新を待たずにタップ即反映が実現できます。

---

## useOptimistic の内部動作

`useOptimistic` は内部的に **キュー構造**を使って楽観的更新を管理しています。`addOptimisticPitches(pitch)` を呼ぶたびに、`pitch`（更新関数への引数）がキューに積まれます。

レンダリングのたびに、React はこう計算します。

```
optimisticPitches = pitches（本物）
                  + キューに残っている楽観的更新を順番に適用
```

`setPitches` が呼ばれると `pitches` が更新され、そのレンダリングで残っているキューが本物のstateの上に乗せ直されます。すべての非同期処理が完了してキューが空になると、`optimisticPitches === pitches` になります。

なお、**管理の単位は個々の非同期処理ではなく、後述する `startTransition` のブロック全体**です。transition内のすべての処理が完了（`setPitches` が呼ばれるか、エラーがthrowされる）して初めてそのtransitionによって積まれたキューが破棄されます。

---

## useTransition が必要な理由

最初、`addOptimisticPitches` を呼んだらこのエラーが出ました。

```
An optimistic state update occurred outside a transition or action.
```

`addOptimisticPitches` の呼び出しは、React の「transition」の中でなければなりません。

React には更新の優先度の概念があります。

- **緊急な更新**：タップへの即反応（遅らせるとUIがフリーズして見える）
- **緊急でない更新**：DB通信・重い計算（多少遅れても体験を損なわない）

`useOptimistic` は「緊急でない更新」が完了するのを待たずにUIを更新するためのツールであり、「その裏で非同期処理が走っているときだけ意味がある」という前提で作られているため、`addOptimisticPitches` は「緊急でない更新の文脈」の中でしか呼び出せません。

`startTransition` がその文脈を作る関数です。`startTransition`で楽観的更新の処理をラップします。

```js
const [isPending, startTransition] = useTransition();
```

---

## 実装の全体像

```js
const handleZoneSelect = async (zoneNumber, gesture) => {
  const result = zoneNumber <= 9 ? "strike_looking" : "ball";

  if (gesture === "tap") {
    startTransition(async () => {
      // ① 仮のstateを即更新 → カウントが即反映される
      addOptimisticPitches({ zone: zoneNumber, result, pitchType: selectedPitchType });

      // ② DB保存（非同期）
      const pitchId = await recordPitch({ ... });

      // ③ 本物のstateを更新 → optimisticPitches が本物に差し替わる
      setPitches([...pitches, { id: pitchId, zone: zoneNumber, result, pitchType: selectedPitchType }]);
    });
  }
};
```

タップ時の流れはこうなります。

```
タップ
 ↓
addOptimisticPitches → optimisticPitches が即更新 → カウントが即反映
 ↓（並行して）
recordPitch（DB保存）
 ↓（完了）
setPitches → optimisticPitches が本物のstateに差し替わる
```

③のタイミングで画面上の表示は変わりません。すでに楽観的UIが正しい状態を表示しているためです。

---

## エラー時の挙動

`recordPitch` がエラーをthrowした場合、`setPitches` が呼ばれません。このとき React が自動で `optimisticPitches` を元の `pitches` に巻き戻します。楽観的UIが「なかったこと」になる仕組みが自動で組み込まれています。

---

## タップ連打時の注意点

transition が完了する前に次のタップが来て新しい transition が始まると、楽観的更新が重複表示されることがあります。最終的には正しい状態に収束しますが、途中でおかしな表示になるケースがあるようです。

### なぜ起きるか（わかっている範囲）

まず前提として、`useOptimistic` の管理単位は個々の非同期処理ではなく **`startTransition` のブロック全体**です。transition内のすべての処理が完了して初めてキューのエントリが破棄されます。

その上で、`useOptimistic` は内部的にpendingな楽観的更新をキューとして積みます。

```
actualState + [tap1のoptimistic] + [tap2のoptimistic]
```

tap1のtransitionが完了すると、キューを本物のstateの上に乗せ直します。

```
新しいactualState（tap1確定済み） + [tap2のoptimistic]
```

tap2のoptimisticは消えません。この再計算の過程で重複表示が起きることがありますが、**なぜ起きるかの詳細なメカニズム（ReactのLane管理と巻き戻し処理の関係）についてはまだ正確に把握できていません**。ReactのLane・Fiber・useTransitionの内部実装まで踏み込む必要があるようです。

なお、今作っている投球記録アプリに当てはめて考えると、野球には一定の投球間隔が存在するため、連打するシチュエーションは少ないと思われます。

---

## まとめ

| | 役割 |
|---|---|
| `useOptimistic` | transition内で仮のstateを即更新してUIを先に進める |
| `useTransition` | 「これは緊急でない非同期処理だ」と React に伝える |
| 2つがセットで必要な理由 | `addOptimisticPitches` の呼び出しはtransition内でなければならないため |

DB通信を伴うタップ系のUIには積極的に使っていきたいパターンです。ただしエラー時のユーザーへの通知はセットで実装するのが安全です。
