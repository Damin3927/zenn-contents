---
title: "SolanaのEscrow実装でつまずいた人へ送る有意義な情報"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Solana", "Anchor"]
published: false
---

## TL;DR

これがソースコードです．
@[card](https://github.com/Damin3927/escrow-anchor)

[この実装](https://hackmd.io/@ironaddicteddog/anchor_example_escrow)を大いに参考にさせていただいたのですが，

1. 一部書き方が古いところなどあったので，全て現行の Anchor のバージョン（0.24.2）の書き方に直した
1. モジュール分割をした
1. できるだけたくさんのコメントを書き加え，「？」なコードがないようにした

という改造を加えているので，とてもわかりやすくなっているかと思います．

## Anchor を学ぶ意義

### Anchor なしの世界

自分で Anchor を触ってて思うのですが，やっぱりフレームワークって大事だなって思います．
最初に Solana のプログラム(Anchor なし)を Rust で書いたときには，「えっ！データのシリアライズ・デシリアライズ関数まで自分で用意するの！？」と誰しも驚くことです．だって Solidity とかでそんなの書いたことないもん()
（Solidity は [ABI(Application Binary Interface)](https://ethereum.org/en/developers/docs/smart-contracts/compiling/#web-applications)という IDL 的存在があって，これがとてもわかりやすく Json で書かれているんですよね．Solana ネイティブのプログラムはこの ABI 生成くらいから読み込みまでまあ自分でやれやみたいな感じです）

今 Solidity という言葉が出てきましたが，Solidity はかなり言語的に高級です．[Yul](https://solidity-jp.readthedocs.io/ja/latest/yul.html) という inline assembly が書けるとか書けないとかそういう話をしているのではありません．プログラマが知らなくていいところをいい感じに全部 EVM に入れ込んでしまっていて，簡単なプログラムであれば（ほぼ）ビジネスロジックのみでスマコンが完成してしまうのです．本当に Solidity はすごいとおもいます．

さて，Solana に戻ってみましょう．Solidity を学習したあとに Solana の[SPL Token 実装](https://github.com/solana-labs/solana-program-library/blob/9e029349fca867dc5a23fa8e571ce95da44976b5/token/program-2022/src/processor.rs#L1115)でもみてみましょう．あっと驚くことには，

1. まず触る可能性のあるアカウントを全て受け取る
1. データを unpack（デシリアライズ的な）し，個々の instruction の enum に match させる
1. その enum に対して当該の processor に処理を渡す
1. 処理中にアカウントへの保存・アカウントからの読出し処理が挟まる場合は，よしなに pack・unpack する
1. イテレータを**手動**で大量の`next_account_info(account_info_iter)?`で読み出し，何番目の account が何に対応していたか忘れるのでコードを行ったり来たりする
1. しまいにはアカウントのバリデーションとビジネスロジックがぐちゃぐちゃに入り混じっており，アーキテクチャ的によくない

> ええぇぇぇぇ〜〜〜〜こんなのプログラマの仕事じゃないよ〜〜〜〜

そうです．誰も生の`data: &[u8]`を触りたいなんて思っていないはずです（変態は知りませんが）．これくらいのことなら全てフレームワークに任せられるはず！

### Anchor の良さ

ってことで Anchor です．Anchor は Anchor なりに既存の Solana のデータ剥き出し状態をうまく隠してくれています．さっきの問題を一つ一つ解決していくと，

1. 触る可能性のあるアカウントは受け取る（ここは Solana 高速化のための並列化を可能にするものなのでやむなし）
1. ただの関数定義から instruction を自動生成する
1. 自動生成された instruction の関数に勝手にルーティングされる
1. アカウントデータには全て「型」が存在しているので，型経由で読出し・保存をする
1. `#[derive(Accounts)]`をもつ構造体で使用するアカウントを定義する．`next_account_info(account_info_iter)?`は使わない
1. アカウントバリデーションは全て`#[derive(Accounts)]`構造体の中で行うので，instruction の中身と混じることがない

こうでなくちゃですよ．Anchor のおかげでボイラープレートは少なくなり，かつコードの責任の分離もとてもはっきりさせることができます．ここでやっと Solidity くらいの高級さに追いついたか，という感じです．

でも少しだけ気になる点があって，それは，Anchor はめちゃくちゃ Rust のマクロを使っています．そのため，少しだけ rust-analyzer の型補完が効かないときがあることです．（もちろんコンパイル時はちゃんとしてくれます）

## Anchor での Escrow 実装までに必要な知識と道のり

それでは実際にどうやって書けるようになればいいのか，というのを順に追っていきます．Ethereum くらいは知っているけど，Solana は初心者だよーという前提です．

### 1. Solana の基本データモデルを理解

@[card](https://docs.solana.com/developing/programming-model/overview)

ここで挙げられている 4 つのセクションは超大事で，これを知らないと何もコードが書けない（そもそも Solana についていけない）ので，ここは是非とも読んでください．一番大事なのは，

1. Solana 上でのほぼ全ての情報は，Account によって管理される
   - Program(Ethereum でいう Smart Contract)も Account に乗っているデータ
1. 全ての Account には **owner** が存在しており，owner のみが data を書き換えられる
1. Transaction には，Transaction によって呼ばれる Program が触る可能性のある Account を全て渡す必要がある

みたいなところだと思います．これは CPI（Cross-Program Invocation）や PDA（Program Derived Account）などを理解するときにもとても役立ちます．

### 2. Anchor なしで Program を書いてみる

次にやるべきことは，ネイティブ Solana で Program を作ることです．なぜこれが必要なのかというと，めちゃ重要な [SPL Token](https://github.com/solana-labs/solana-program-library/tree/master/token/program-2022) や [Associated Token Account](https://github.com/solana-labs/solana-program-library/tree/master/associated-token-account/program) の実装も読めないし，現在結構イケてる[mango-v3](https://github.com/blockworks-foundation/mango-v3)などのコードも読めなくなるからです．世の中にはまだまだ Solana ネイティブな実装で走っている Program が多く，それを理解するのはとても大事です．

題材としてお薦めするのは，Solana での Escrow 実装を解説している<https://paulx.dev/blog/2021/01/14/programming-on-solana-an-introduction/> です．8 割理解で全然構わないと思うので，写経をお勧めします．

### 3. Anchor で Program を書く

最後に，Anchor で Program を書きます．2 をクリアすると Anchor の公式 docs (二種類ある．[旧](https://project-serum.github.io/anchor/)と，[Anchor Book](https://book.anchor-lang.com/)があり，今は改修中で片方にしかない情報がある)は簡単に読めるかと思います．逆にいうと，Solana でネイティブで書いたことがないまま急に Anchor の勉強しても何もわからないと思います．

ここでもお勧めなのが，Escrow の Anchor 実装です．

@[card](https://github.com/Damin3927/escrow-anchor)

[この実装](https://github.com/ironaddicteddog/anchor-escrow)と[その解説記事](https://hackmd.io/@ironaddicteddog/anchor_example_escrow)を参考にしたのですが，書き方が少し古いのと，モジュール分割がされていないなどの理由から新しく実装をしました．加えて，たくさんのコメントを付け加えたため，わかりやすくはなっているかと思います．

## Conclusion

Solana はイケてます．ただし開発者もおっしゃっていますが，ドキュメントより開発を進めることを大事にしているようなので，ドキュメントがそんなに多くありません．（特に client 側の sdk はなさすぎて内部コード読むしかありません．）

誰かの役に立つことを願っています．
