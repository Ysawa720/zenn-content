---
title: "useOptimistic を導入して、結局外した話 — 手動 state 管理との二重カウント"
emoji: "⚾"
type: "tech"
topics: ["react", "nextjs", "useoptimistic", "frontend"]
published: true
---

## はじめに

前回、草野球の投球記録アプリで `useOptimistic` + `useTransition` を使ってタップ即反映UIを作った話を書きました。

https://zenn.dev/ysaya_dev/articles/react-useoptimistic

その記事で「DB通信を伴うタップ系のUIには積極的に使っていきたいパターン」と締めくくったのですが、その後この `useOptimistic` を**外しました**。

導入してから外すまでに、カウント表示のチラつきという形でバグが現れ、原因を追ううちに「そもそもこのアプリの構成に `useOptimistic` が合っていなかった」という結論にたどり着きました。今回はその顛末をまとめます。

前回の記事で説明したキュー構造や transition 単位の管理は前提として進めるので、そのあたりが気になる方は先に前編を読んでもらえるとスムーズです。

## 結論（3行）

- `useOptimistic` は Server Actions + `revalidatePath` でデータが再取得される構成を前提に設計されている
- Client Component で base state を手動 `setState` する構成だと、楽観エントリと実データが一瞬重複して二重カウントが起きる
- このアプリはローカルファースト（IndexedDB）に進む予定で、そうなると `useOptimistic` が解決したかった遅延がそもそも消える

## 症状：カウントがチカチカする

前回の実装を入れた直後から、カウント表示がチカチカするようになりました。

最初に立てた仮説は「楽観的更新と実際の更新で、同じ処理が2回反映されているのでは」というものでした。けれど、`useOptimistic` が正しく動いていれば楽観エントリと実データのカウントは同じ値になるはずで、値が同じならチラつく理由がありません。

ここで詰まったのですが、画面をよく観察すると遷移の中身が見えてきました。0ストライクの状態でストライクをタップしたとき、こう動いていたのです。

```
0ストライク → 1ストライク → 2ストライク → 1ストライク
```

一瞬だけ「2ストライク」が挟まっている。チラつきの正体はこれでした。最終的に1に戻るということは、実データには1球分しか入っていないのに、途中で2球分にカウントされる瞬間があるということです。

## 原因：base state を transition 中に更新していた

問題のコードはこうなっていました。

```jsx
const [pitches, setPitches] = useState([]);
const [optimisticPitches, addOptimisticPitches] = useOptimistic(
  pitches,
  (state, newPitch) => [...state, newPitch]
);

const handleZoneSelect = (zoneNumber) => {
  startTransition(async () => {
    addOptimisticPitches(newPitch);       // ① 楽観エントリ追加
    const pitchId = await recordPitch(...); // ② DB保存
    setPitches([...pitches, realPitch]);   // ③ base state も更新
  });
};
```

前回の記事で書いたとおり、`useOptimistic` は **base state（第1引数の `pitches`）** と **まだ pending な楽観エントリのキュー** を合成して値を返します。キューが破棄されるのは transition のブロック全体が完了したときです。

ここで問題なのが③の `setPitches` です。これは `useOptimistic` の base state を、**transition がまだ完了していないうちに手動で増やしている**ことになります。

順を追うとこうなります。

```jsx
startTransition(async () => {
  addOptimisticPitches(newPitch);   // ① 楽観エントリ追加 → 表示は1球
  await recordPitch(...);
  setPitches([...pitches, realPitch]); // ② base state が1球に増える
  //                                       でもキューの楽観エントリはまだ残っている
  //                                    → base(1球) + キュー(1球) = 2球!
  // ③ transition 完了 → キュー破棄 → base(1球)だけ → 1球に戻る
});
```

`setPitches` を呼んだ瞬間、base state が1球になります。でも transition はまだ完了していない（関数の末尾まで実行されていない）ので、楽観エントリのキューも残ったままです。`useOptimistic` はこの2つを合成するので、一瞬だけ2球分になる。これが「2ストライク」の正体でした。

`0 → 1 → 2 → 1` はこうして生まれていたわけです。

ポイントは、**`useOptimistic` の base state を transition 完了前に自分で触ったこと**です。`useOptimistic` は「base state はいずれ外側から更新される」前提で作られているので、自分で base を増やすと楽観エントリと二重になります。

## 解決策を検討する

### 案1: tempId で「追加」を「置き換え」にする

楽観エントリに一時的な ID を持たせ、実データが来たときに追加ではなく置き換えにすれば二重カウントは消えます。

```jsx
const [optimisticPitches, addOptimisticPitches] = useOptimistic(
  pitches,
  (state, newPitch) => {
    const exists = state.some((p) => p.tempId === newPitch.tempId);
    if (exists) {
      return state.map((p) => (p.tempId === newPitch.tempId ? newPitch : p));
    }
    return [...state, newPitch];
  }
);
```

ここで「`setPitches` 側で `pitchId`（サーバー採番のID）を付けているのに、なぜ別途 `tempId` が必要なのか」と疑問に思うかもしれません。

理由は、**楽観エントリと実データを突き合わせられる共通の識別子が `tempId` しかない**からです。

問題が起きるのは `setPitches` が呼ばれた瞬間で、このとき `useOptimistic` が updater を再実行します。

```jsx
optimisticPitches = updater(新しい実データ, まだ pending の楽観エントリ)
// state    = 新しい実データ = [{ id: pitchId, tempId: 'abc', ... }]
// newPitch = pending の楽観エントリ = { tempId: 'abc', ... }
```

`pitchId` はサーバーが処理を終えて初めて返ってくる値です。`addOptimisticPitches` を呼ぶ時点ではまだ存在しないので、**楽観エントリ側に `pitchId` を含められません**。つまり `pitchId` では両者を一致させられない。

一方 `tempId` はサーバー処理の前にクライアントで生成するので、楽観エントリと実データの両方に持たせられます。これが唯一の共通識別子になります。

updater の中では、この `tempId` で一致を判定しています。

```jsx
const exists = state.some((p) => p.tempId === newPitch.tempId);
// 'abc' === 'abc' → true

if (exists) {
  return state.map((p) => (p.tempId === newPitch.tempId ? newPitch : p));
  // 実データ内の tempId:'abc' のエントリを楽観エントリで「置き換え」
  // 配列の長さは変わらない → 1球分のまま
}
return [...state, newPitch]; // ここには到達しない
```

`exists` が `true` なので `return [...state, newPitch]`（追加）には到達せず、置き換えになります。配列の長さが変わらないので二重カウントが起きません。置き換えといっても内容はほぼ同じ（`id` の有無が違うだけ）なので、表示上の変化もカウントの変化もありません。

技術的にはこれで直ります。ただ、対症療法であって構造的な解決ではありません。本筋は「base state を transition 中に触らない」ことのはずです。

### 案2: そもそも useOptimistic に向いている構成か

ここで一度立ち止まって、`useOptimistic` の想定される使い方を確認しました。

`useOptimistic` は Server Components + Server Actions + `revalidatePath` のセットで使うことを想定して設計されています。本来の流れはこうです。

```
addOptimistic(newItem)   ← 楽観表示（即時）
await serverAction()     ← DB書き込み + revalidatePath() を内部で呼ぶ
                         ← Next.js がページの Server Component を再実行
                         ← 最新データが props 経由で降ってくる
                         ← base state が外側から更新される
（自分で setState は呼ばない）
```

この構成だと base state を手動で `setState` しないので、そもそも二重カウントが起きる余地がありません。

つまり今回のバグは、`useOptimistic` を Client Component で `useState` の手動管理と組み合わせたことの構造的な副作用でした。`useOptimistic` を Client Component で使うこと自体は普通ですが、確定データの更新までサーバーに任せられない構成だと、base state の整合性を自分で取る必要が出てきて、今回のような問題が起きやすくなります。

## さらに：ローカルファーストとの相性

もうひとつ、設計の方向性として効いてきた点があります。

このアプリは将来、オフライン対応（IndexedDB / Dexie.js）を予定しています。試合中のグラウンドは電波が不安定なことが多く、ローカルに先に書いてから後でサーバーに同期する構成にしたいからです。

ここで `useOptimistic` との前提のズレが効いてきます。

| | useOptimistic の前提 | ローカルファーストの前提 |
| --- | --- | --- |
| 真実の場所 | サーバー | ローカル DB |
| 書き込み確定のタイミング | サーバーが応答したとき | ローカル DB に書いた瞬間 |
| 遅延 | 数百 ms | ほぼ 0 ms |

ローカル DB への書き込みは即時かつほぼ確実に成功します。つまり「サーバーが確認するまで仮の値を見せておく」という `useOptimistic` の役割そのものが不要になります。前編で `useOptimistic` を入れて解消した「タップから反映までの遅延」が、ローカルファーストにした時点で根本から消えるのです。

## 結論：いったん外す

以上を踏まえて、今の段階で tempId の複雑な実装を入れて `useOptimistic` を使い続けるより、一度外して手動 `setState` だけにする方が合理的だと判断しました。

- tempId 案は対症療法で、構造的な問題は残る
- そもそもこのアプリの構成（Client Component + 手動 state 管理）は `useOptimistic` の想定と合っていない
- 遅延はオフライン対応を入れる段階で自然に解消される

前編で「積極的に使っていきたいパターン」と書いたものを、数週間で外すことになりました。少し気恥ずかしいですが、これはこれで学びの多い回り道でした。

## 教訓

- `useOptimistic` は設計と合ったアーキテクチャで使う
  - Client Component で `useOptimistic` を使い、確定データの更新は Server Action + `revalidatePath` に任せる → 相性が良い
  - Client Component で base state を手動 `setState` 管理する → 二重カウントなど整合性の問題が起きやすい
  - ローカルファースト → そもそも不要になることがある
- 「タップ即反映」を実現する手段として `useOptimistic` を選んだが、アーキテクチャ全体を見渡すと別の手段（ローカル DB）の方が自然に問題を解決できる場合がある
- ツールの使い方だけでなく、そのツールがどういう設計思想に基づいているかを理解してから導入することが大事だと改めて感じた

前編で仕組みをかなり丁寧に追ったぶん、「仕組みは理解していても、構成と噛み合っていなければ問題は起きる」というのが今回の一番の収穫でした。
