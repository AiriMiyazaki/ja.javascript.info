# ガベージコレクション

JavaScriptのメモリ管理は、自動で私たちの目には見えないように行われます。私たちが作るプリミティブ、オブジェクト、関数... それらはすべてメモリを必要とします。

何かがもう必要なくなったとき、何が起こるでしょう？JavaScriptエンジンはどのようにそれを検出し、クリーンアップするのでしょうか？

<<<<<<< HEAD:1-js/04-object-basics/02-garbage-collection/article.md
[cut]

## 到達性 
=======
## Reachability
>>>>>>> fe571b36ed9e225f29239e82947005b08d74ac05:1-js/04-object-basics/03-garbage-collection/article.md

JavaScriptのメモリ管理の主要なコンセプトは、*到達性* です。

簡単に言えば、「到達可能な」値は、何らかの形でアクセス可能、または使用可能な値です。それらはメモリに格納されることが保証されています。

1. 本質的に到達可能な値の基本セットがあり、それらは明白な理由により削除されません。

    例:

    - 現在の関数のローカル変数とパラメータ
    - ネストされた呼び出しの、現在のチェーン上の他の関数のローカル変数とパラメータ
    - グローバル変数
    - (他にも幾つか同様に内部のものがあります)

    それらの値は *ルート* と呼ばれます。

2. 他の任意の値は、参照または参照のチェーンにより、ルートから到達可能であれば、到達可能とみなされます。

<<<<<<< HEAD:1-js/04-object-basics/02-garbage-collection/article.md
    例えば、ローカル変数にオブジェクトがあった場合、そしてそのオブジェクトが別のオブジェクトの参照をもっていたとすると、そのオブジェクトは到達可能とみなされます。そして、それが参照するものもまた到達可能です。詳しくは後述します。
=======
    For instance, if there's an object in a global variable, and that object has a property referencing another object, that object is considered reachable. And those that it references are also reachable. Detailed examples to follow.
>>>>>>> fe571b36ed9e225f29239e82947005b08d74ac05:1-js/04-object-basics/03-garbage-collection/article.md


JavaScriptエンジンには[ガベージコレクタ](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))と呼ばれるバックグラウンドプロセスがあります。それはすべてのオブジェクトを監視し、到達不可能になったオブジェクトを削除します。

## シンプルな例 

これは最もシンプルな例です:

```js
// user は オブジェクトへの参照を持っています
let user = {
  name: "John"
};
```

![](memory-user-john.svg)

ここで、矢印はオブジェクトの参照を示しています。グローバル変数 `user` はオブジェクト `{name: "John"}` を参照しています(簡潔のためにそれを John と呼びます)。John の `"name"` プロパティはプリミティブを格納しているので、オブジェクトの枠内に描かれています。

もし `user` の値が上書きされると、参照はなくなります。:

```js
user = null;
```

![](memory-user-john-lost.svg)

今、John は到達不可能になりました。そこへの参照がないため、アクセスする手段はありません。ガベージコレクタはデータを捨てて、メモリを解放します。

## 2つの参照 

さて、`user` の参照を `admin` にコピーしたとしましょう。:

```js
// user は オブジェクトへの参照を持っています
let user = {
  name: "John"
};

*!*
let admin = user;
*/!*
```

![](memory-user-john-admin.svg)

今、先ほどと同じことをしたとすると:
```js
user = null;
```

...オブジェクトはまだグローバル変数 `admin` 経由で到達可能です。なのでメモリ上にいます。もし `admin` も上書きしたら、それは削除されます。

## 連結されたオブジェクト 

これはより複雑な例です。家族(family):

```js
function marry(man, woman) {
  woman.husband = man;
  man.wife = woman;

  return {
    father: man,
    mother: woman
  }
}

let family = marry({
  name: "John"
}, {
  name: "Ann"
});
```

関数 `marry` は与えられた2つのオブジェクトを互いに参照することによって "marry(結婚)" し、それら両方を含む新しいオブジェクトを返します。

結果のメモリ構造:

![](family.svg)

今のところ、全てのオブジェクトは到達可能です:

ここで、2つの参照を削除してみましょう:

```js
delete family.father;
delete family.mother.husband;
```

![](family-delete-refs.svg)

それら2つの参照のうち、1つだけを削除するのでは不十分です。なぜなら全てのオブジェクトはまだ到達可能だからです。

しかし、もし両方を削除すると、John にはもう参照がないことがわかります:

![](family-no-father.svg)

外への参照は気にする必要はありません。内への参照のみがオブジェクトを到達可能にします。なので、John は今や到達不可能で、到達不可能になったその全てのデータとともにメモリ上から削除されるでしょう。

ガベージコレクションの後:

![](family-no-father-2.svg)

## 到達不可能な島 

連携されたオブジェクトの島全体が到達不可能になり、メモリから削除される場合があります。

ソースとなるオブジェクトは上記と同じです。 次にこのようにします：

```js
family = null;
```

すると、メモリの中はこうなります:

![](family-no-family.svg)

この例は、到達可能性の概念がどれほど重要であるかを示しています。

John と Ann はまだリンクされているのは明らかです。両方は内への参照を持っています。しかしそれだけでは十分ではありません。

前者 `"family"` オブジェクトはルートからのリンクが解除されており、これ以上の参照はないので、島全体が到達不可能になり、削除されます。

## 内部のアルゴリズム 

基本のガベージコレクションのアルゴリズムは "マーク・アンド・スイープ" と呼ばれます。

次の "ガベージコレクション" のステップは定期的に実行されます:

<<<<<<< HEAD:1-js/04-object-basics/02-garbage-collection/article.md
- ガベージコレクタはルートを取得し、 それらを "マーク" します(覚えます)。
- 次に、そこからの全ての参照へ訪れ、"マーク" します。
- 次に、マークされたオブジェクトへアクセスし、*それらの* 参照をマークします。訪問されたすべてのオブジェクトは記憶されているので、将来同じオブジェクトへは2度訪問しないようにします。
- ...そしてまだ訪れていない参照(ルートから到達可能)があるまで行います。
- マークされたオブジェクトを除いた、すべてのオブジェクトが削除されます。
=======
- The garbage collector takes roots and "marks" (remembers) them.
- Then it visits and "marks" all references from them.
- Then it visits marked objects and marks *their* references. All visited objects are remembered, so as not to visit the same object twice in the future.
- ...And so on until every reachable (from the roots) references are visited.
- All objects except marked ones are removed.
>>>>>>> fe571b36ed9e225f29239e82947005b08d74ac05:1-js/04-object-basics/03-garbage-collection/article.md

例えば、我々のオブジェクト構造はこのように見えます:

![](garbage-collection-1.svg)

右側に明らかに "到達不可能な島" が見えます。今、どのように "マーク・アンド・スイープ" ガベージコレクタがそれを扱うか見てみましょう。

最初のステップではルートをマークします:

![](garbage-collection-2.svg)

次に、それらの参照をマークします:

![](garbage-collection-3.svg)

...そしてそれらの参照も、可能な限り、マークしていきます:

![](garbage-collection-4.svg)

この手順でたどりつけなかったオブジェクトは到達不能とみなされ、削除されます:

![](garbage-collection-5.svg)

<<<<<<< HEAD:1-js/04-object-basics/02-garbage-collection/article.md
ガベージコレクションは、このような考え方で実行されます。

JavaScriptエンジンは多くの最適化を適用して実行を高速化し、実行に影響を与えません。
=======
We can also imagine the process as spilling a huge bucket of paint from the roots, that flows through all references and marks all reachable objects. The unmarked ones are then removed.

That's the concept of how garbage collection works. JavaScript engines apply many optimizations to make it run faster and not affect the execution.
>>>>>>> fe571b36ed9e225f29239e82947005b08d74ac05:1-js/04-object-basics/03-garbage-collection/article.md

これらは最適化のいくつかです:

- **世代コレクション** -- オブジェクトは2つのセットに分割されます: "新しいオブジェクト" と "古いオブジェクト" です。多くのオブジェクトは出現し、仕事をするとすぐ不要になります。これらは積極的にクリーンアップされます。長期間生き残ったものは "古い" ものになり、それほど頻繁には検査されません。
- **インクリメンタルコレクション** -- 多くのオブジェクトがある場合、1度に全てのオブジェクトの集合をマークしようとすると、時間がかかってしまい、実行時に目に見える遅延を引き起こす可能性があります。なので、エンジンはガベージコレクションを小さく分割します。そしてそれらのピースが1つずつ、別々に実行されます。変更を追跡するため、多少の追加の記憶域を必要とはしますが、大きな遅延ではなく多くの小さな遅延になります。
- **アイドルタイムコレクション** -- ガベージコレクタは、CPUがアイドル状態のときにのみ実行を試み、実行への影響を減らします。

<<<<<<< HEAD:1-js/04-object-basics/02-garbage-collection/article.md
ガベージコレクションアルゴリズムには他にも最適化や加減があります。ここでそれらも説明したいのですが、止めておかなくてはいけません。なぜなら、エンジンによって、調整方法やテクニックの使い方がバラバラだからです。 そして、さらに重要なのは、エンジンの開発に伴って状況が変化するため、実際に必要がない場合には「先立って」深く進んでいくことはそれほど価値はありません。 もちろん、それが純粋な興味であれば、参照すると良いいくつかのリンクが下にあります。
=======
There exist other optimizations and flavours of garbage collection algorithms. As much as I'd like to describe them here, I have to hold off, because different engines implement different tweaks and techniques. And, what's even more important, things change as engines develop, so studying deeper "in advance", without a real need is probably not worth that. Unless, of course, it is a matter of pure interest, then there will be some links for you below.
>>>>>>> fe571b36ed9e225f29239e82947005b08d74ac05:1-js/04-object-basics/03-garbage-collection/article.md

## サマリ 

知っておくべきポイント:

- ガベージコレクションは自動で実行されます。実行を強制したり、防ぐことはできません。
- オブジェクトは到達可能な間、メモリ上に保持されます。
- 参照されることは、(ルートから)到達可能であることと同じではありません: 連結されたオブジェクトのパックはそれ全体で到達不可能になる可能性があります。

現代のエンジンはガベージコレクションの高度なアルゴリズムを実装しています。

一般的な本 "ガベージコレクションハンドブック": 自動メモリ管理の技術（R.Jones et al）は、それらのいくつかをカバーしています。

もしあなたが低レベルのプログラミングに慣れている場合、V8のガベージコレクションに関するより詳細な情報はこの記事[A tour of V8: Garbage Collection](http://jayconrod.com/posts/55/a-tour-of-v8-garbage-collection).にあります。

<<<<<<< HEAD:1-js/04-object-basics/02-garbage-collection/article.md
[V8 blog](http://v8project.blogspot.com/) には、メモリ管理の変更に関する記事も随時掲載されています。 ガベージコレクションを学ぶには、V8の内部について一般的に学習し、V8エンジニアの一人として働いていた [Vyacheslav Egorov](http://mrale.ph) のブログを読むとよいでしょう。 "V8", それはインターネット上の記事で最もよくカバーされるからです。 他のエンジンでは、多くのアプローチは似ていますが、ガベージコレクションは多くの点で異なります。
=======
[V8 blog](https://v8.dev/) also publishes articles about changes in memory management from time to time. Naturally, to learn the garbage collection, you'd better prepare by learning about V8 internals in general and read the blog of [Vyacheslav Egorov](http://mrale.ph) who worked as one of V8 engineers. I'm saying: "V8", because it is best covered with articles in the internet. For other engines, many approaches are similar, but garbage collection differs in many aspects.
>>>>>>> fe571b36ed9e225f29239e82947005b08d74ac05:1-js/04-object-basics/03-garbage-collection/article.md

低レベルの最適化が必要な場合は、エンジンに関する深い知識が必要です。 あなたが言語に精通した後の次のステップとしてそれを計画することが賢明でしょう。