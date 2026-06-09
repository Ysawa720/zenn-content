title: "JavaScriptのthisが分からなくなる原因は「外側」の意味だった"
emoji: "🎯"
type: "tech"
topics: ["javascript", "this", "アロー関数", "初心者"]
published: false

はじめに
JavaScript を学んでいて、多くの人がつまずくのが this です。私自身、setTimeout の中で this.name が undefined になる定番のバグに出会い、「なぜアロー関数で直るのか」をきちんと言葉にできずモヤモヤしていました。
調べて整理していくと、this の挙動は たった2つの原則 に集約できることが分かりました。さらにアロー関数の this でつまずく最大の原因は、「外側」という言葉の意味を誤解していることだ、という結論にたどり着きました。
この記事では、その2つの原則を軸に、setTimeout の定番バグがなぜ起きてなぜ直るのかまでを一本につなげて整理します。
大前提：this は関数の中で使うもの
まず押さえておきたいのは、this は関数の中で登場するという点です。
this 単体が独立して存在しているわけではありません。「このオブジェクトの this」のような言い方をついしてしまいますが、これは正確ではありません。this は 関数が実行されるときに、その関数の中で中身が決まるものです。
javascriptconst obj = {
  name: 'JS',
  show() {
    console.log(this.name); // ← thisは「showという関数の中」で意味を持つ
  }
};
オブジェクト obj 自体が this を持っているのではなく、show という関数が実行されたときに、その中の this が決まる。この感覚が出発点になります。
原則①：通常の関数の this は「呼び出し方」で決まる
通常の関数（function で書く関数やメソッド）は、this という箱を持っていますが、中身は空です。中身は 呼ばれた瞬間に、呼び出し方によって決まります。
同じ関数でも、呼び方が違えば this の中身が変わります。
javascriptfunction show() {
  console.log(this);
}
メソッドとして呼ぶ → ドットの左側
javascriptconst obj = { show };
obj.show(); // this = obj
単独で呼ぶ → グローバル（strictモードでは undefined）
javascriptshow(); // this = グローバル or undefined
new で呼ぶ → 新しく作られるインスタンス
javascriptnew show(); // this = 生成された新しいオブジェクト
call で明示的に指定する → 指定したもの
javascriptshow.call({ x: 1 }); // this = { x: 1 }
call は、関数を呼び出すときに this の中身を自分で明示的に指定するためのメソッドです。ドットの左側や new に頼らず、this に何を入れるかを直接手で渡せます。call の 第1引数がそのまま this になるのがポイントです。
javascriptfunction show() {
  console.log(this.x);
}

show.call({ x: 1 });  // 1  ← thisに { x: 1 } を入れて実行
show.call({ x: 99 }); // 99 ← thisに { x: 99 } を入れて実行
第2引数以降は、関数に渡す通常の引数になります。
javascriptfunction greet(greeting) {
  console.log(greeting + ', ' + this.name);
}

greet.call({ name: 'JS' }, 'Hello'); // "Hello, JS"
//          ↑ thisに入る       ↑ 第1引数(greeting)
同じ show という関数なのに、this が4通りに変わります。ここが「定義した場所ではなく、呼ぶ側が決める」という意味です。call はその中でも、this を手動で明示指定できる唯一の呼び出し方だと言えます。
obj.show() は show.call(obj) とほぼ同じことをしています。普段ドット記法が自動でやってくれる「this の指定」を、call は手動でやっているだけ、という関係です。
原則②：アロー関数は this を持たず、外側から借りる
アロー関数は、通常の関数とまったく違います。this の箱をそもそも持っていません。
では this と書いたらどうなるか。自分の外側の this を借りてきます。
javascriptdelayLog() {
  setTimeout(() => {
    console.log(this.name); // 外側のthisを借りる
  }, 1000);
}
ここで重要なのが「外側」の意味です。多くの人（私も）がここを誤解します。
つまずきの核心：「外側」は実行の流れではなく「書かれた位置」
アロー関数が見にいく「外側」とは、そのアロー関数がソースコード上でどの関数の中に書かれているかです。
「いつ・どこで呼ばれるか」という実行の流れは 一切関係ありません。
たとえば次のコードで「コールバックは setTimeout の中で実行されるんだから、setTimeout の this を借りるのでは？」と考えたくなります。
javascriptdelayLog() {
  setTimeout(() => {
    console.log(this.name);
  }, 1000);
}
しかし、このアロー関数はコード上 delayLog の波カッコの中 に書かれています。setTimeout(...) の引数として渡してはいますが、書かれている位置は delayLog の内部です。
setTimeout は「このアロー関数を1秒後に実行してね」と受け取って呼び出すだけで、アロー関数のコードを囲んでいるわけではありません。だから外側は setTimeout ではなく、delayLog になります。
この「書かれた位置で決まる」性質を レキシカルスコープ（静的スコープ） と呼びます。アロー関数の this はレキシカルだ、とよく言われますが、意味は「実行の流れではなく、ソースコードの見た目の囲いで this が決まる」ということです。
定番バグを一本につなげる
ここまでの2つの原則で、setTimeout の定番バグがすべて説明できます。
壊れているコード（通常の関数）
javascriptconst obj = {
  name: 'JS',
  delayLog() {
    setTimeout(function() {
      console.log(this.name); // undefined
    }, 1000);
  }
};

obj.delayLog();
setTimeout に渡したのは 通常の関数です。通常の関数は原則①の通り「呼ばれ方」で this が決まります。このコールバックを呼ぶのは setTimeout の仕組みで、単独呼び出し扱いになるため this はグローバル（または undefined）になります。
結果、this.name が undefined になります。
直したコード（アロー関数）
javascriptconst obj = {
  name: 'JS',
  delayLog() {
    setTimeout(() => {
      console.log(this.name); // 'JS'
    }, 1000);
  }
};

obj.delayLog();
流れを追うとこうなります。

obj.delayLog() と呼ぶ → 原則①より delayLog の中の this は obj
アロー関数はコード上 delayLog の中に書かれている → 外側は delayLog
アロー関数は外側（delayLog）の this を借りる → それは obj
だから this.name は 'JS'

delayLog というメソッドが「this = obj という状態」を作り、アロー関数がそれを受け取る、という二段構えです。
注意：delayLog が無いとどうなるか
「delayLog で囲まなくても、obj の this を借りればいいのでは？」と思うかもしれません。しかし前述の通り、obj 自体は this を持っていません。
javascriptconst obj = {
  name: 'JS',
  cb: () => {
    console.log(this.name);
  }
};

obj.cb(); // undefined
このアロー関数を囲む関数が無いため、外側は obj ではなく、その さらに外＝トップレベル になります。トップレベルの this（ブラウザなら window）に name が無いので undefined です。
this がちゃんと意味を持つ場所（メソッドの中やクラスの中）でアロー関数を書いて初めて、this を正しく借りられる、ということです。
おまけ：グローバルの this は「無い」わけではない
「トップレベルの this って無いの？」という疑問もありますが、実は 環境によって中身が違うだけで、存在はしています。

ブラウザのスクリプト直書き → window
Node.js の CommonJS モジュール → module.exports（空の {}）
ES モジュール → undefined

前述の undefined は「this が無いから」ではなく「this（window など）に name が無いから」です。ES モジュールだと this が undefined なので、undefined.name で エラー になる点にも注意が必要です。同じコードでも環境で結果が変わるのはこのためです。
実用上は、グローバルの this に意図的に頼る場面はあまり多くないように思います。this は「this が意味を持つ場所」で使うもの、くらいに考えておくとよさそうです。
まとめ
this で迷わないための核心は、この2つだけです。

通常の関数の this は「呼び出し方」で決まる（ドットの左、new、call など）
アロー関数の this は「コードを書いた位置の外側」から借りてくる（レキシカル）

そして大前提として、this は 関数の中で実行時に決まるもの であり、オブジェクトが this を持っているわけではない、という点を押さえておくこと。
この3つが腑に落ちると、setTimeout の定番バグはもう怖くありません。
