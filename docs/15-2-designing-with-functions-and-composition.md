# 第二回 : 関数と合成による設計

第一回では、無限になりうるデータ構造を、実際にそのデータ構造を扱う関数から分離するために、遅延評価が役に立つことを見ました。ここでは同じ例を他の角度から復習しましょう。インクリメンタル開発を可能にする、関数を利用した設計についてです。

ここまで我々は、ゲームの盤面からなる木を構築する方法について扱ってきました。このゲーム木において子ノードは、プレイヤーが交互に手を進めることによって生み出される複数の盤面を表現しています。

次に打つべき最善手を見つけるために、盤面に何らかの値を付与しその値が最大値を取る盤面を選べるようにする必要があります。ここは _Double_ 型を採用して、コンピュータの勝利が確定した状態を `1` とし、逆を `-1` としましょう。盤面に対してこの値を静的に返す関数の型は以下のようになります。

Caption: 静的な評価関数

```
static :: Board -> Double
```

今回の議論を進める上では、特定のゲームに関して評価関数をどう実装するかは考える必要はありません。もし興味があれば、○×ゲームに対するシンプルな盤面評価の方法が [リンク先](http://github.com/Dierk/fregePluginApp/blob/game_only/src/frege/fregepluginapp/Minimax.fr) にあります。

Caption: オブジェクト指向設計との違い

ここで少し立ち止まって、もしオブジェクト指向設計だったらどうするかを考えてみましょう。`static` を `Board` 型のメソッドにしたくなるでしょう？ この方法ではメソッドとそれが扱うデータをまとめてカプセル化できるため、特にメソッドが盤面の内部で呼ばれる場合など、何らかの仮定をおいた場合には利点があるでしょう。

欠点としては、盤面をどのように扱うかに注意を向けたとき、それが _侵入的_ な変更となって現れる点が挙げられます。

Frege では、最終的な決定を保留したままであっても、データ構造と関数を同じ _モジュール_ 内に置くことによってカプセル化の利点を維持することができます。

そして次に取れる手について考える際、少なくとも一手読みの場合には、結果として得られる盤面の評価値が最大になるようなものを選ぶことになるでしょう。少し先まで読もうとすればすぐに、対戦相手がどんな行動を取るかも考える必要が生じます。合理的に考えて、対戦相手は逆に盤面の評価値が最低になるような最善手を打つだろうと仮定します。そして相手もこちらが最善手を打つだろうと仮定する、そんな風に進んでいきます。

つまり、盤面の評価値の中でも、木構造の葉が保持している静的な値のみが興味の対象です。それ以外の値はすべて、ひとつ下のレベルのノードが保持している最大・最小値から算出されます。以下の図は、与えられた盤面に対して次の手を計算する様子です。丸が付いている数字は静的な評価値であり、付いていない数字はミニマックス値であることに注意してください。

Caption: 二手読みによるミニマックス木

これを自然に関数型の設計に移すと以下のようになります。

Caption: 相互再帰によるミニマックス法のアルゴリズム

```
maximize :: Ord a => Tree a -> a
maximize (Node a []) = a
maximize (Node a children) = maximum (map minimize children)

minimize :: Ord a => Tree a -> a
minimize (Node a []) = a
minimize (Node a children) = minimum (map maximize children)
```

このやり方は本当に「自然」でしょうか？ 答えは Yes。関数型では、必要最小限の仮定（ホーアが証明した「最弱事前条件」のように）だけを置きます。木構造に対してミニマックス法を適用するために必要なのは、要素が何らかの方法で順序付け可能（Java の用語で言えば「comparable」）であることだけです。型注釈 `Tree a → a` において「a」は型変数で、「a」型の要素を持つ木から全く同じ「a」型の値へのとマッピングが行われることを示しています。「a」型についてここで仮定されているのは制約 `Ord a ⇒` のみ、すなわち、`minimum` と `maximum` を呼び出すために「a」型が `Ord` 型クラスのインスタンスでなければならない、という制約のみです。

次に思いつく疑問として、このような早い段階で抽象化を行うことは普通でしょうか？ この方法では型宣言を完全に消し、Frege に型を推論させることが可能です。いいですか、我々が与えたものと同じ型が正確に推論できるのです！ はたしてこれ以上自然なことがあるでしょうか？

早い段階で抽象化することの恩恵として、完全に一般化されたミニマックス法の解法を得ることができ、その解法は今回扱っているゲーム木とはまったく独立です。ここでも _非侵入的_ な差分を導入したことになります。

しかしこれはうまくいくでしょうか？ 「a」に相当するのは `Board` 型ですが、盤面同士は（まだ）比較可能ではありません。ここで関数合成が再び登場します。関数合成については、枝刈り関数と `gameTree` を合成して`prune 5 . gameTree` の形で枝刈りされた木を作り出した際、すでに見たとおりです。同じようにして、盤面の木から、盤面に対する静的な評価値の木を生成することができます。

Caption: ゲーム木から Double 値の木へ

```
valueTree = static . prunedTree
```

この木はまだ具現化されていないということを忘れないように！ アクセスする必要があるときにのみ、子要素が生成されます。そしてそれゆえに `static` 関数と相性がよいのです。というのも、`static board` を介して盤面にアクセスしない限り、何も評価されることはないのですから。

盤面に対して最大値を計算する関数もまた関数合成です。

```
maximize . valueTree
```

あるいはこれを展開した形である以下のようになります。

```
maxValue = maximize . static . prune 5 . gameTree
```

Caption: 盤面はどこに？

定義の中で、盤面はどこに行ってしまったのかと思うかもしれません。最終的には `maxValue` は盤面を引数に取り `Double` 値を返すはずです。盤面のパラメータはどこにあるのでしょう？ たしかに、ここでパラメータを明示することもできました。しかし上記の定義は、今考えているのが _関数定義_ であって、_関数適用_ ではないということをよく表しています。`maxValue` 関数は、右辺に現れる関数たちの _関数合成_ になっているのです。

「=」記号が表しているのは _定義_ であり、_代入_ ではないことに注意しましょう。純粋関数型プログラミングでは代入を使用することは決してありません。

これで、好きなように合成できる関数が揃いました。コンパイラはすべての型を推論し、関数たちが正しく並んでいることを確認できるようになっています。例えば、順序を定義していない値の木にうっかり`miximize` を適用してしまうことはありえません。しかしこのような木を生成することは可能で、現に、本来 `gameTree` 自体には順序が定義されていません。

型があることによって、`prune` は `static` を呼び出す前でも後でも使えることがわかる一方、動作しない位置に `prune` を置いてしまうようなエラーは検出することができます。

今回の内容は、単に遊んで楽しい道具だというだけではなく、複数の意味で _合成的_ です。ここに至るまでの過程は完全に _非侵入的_ かつ _インクリメンタル_ な方法でなされており、すでに定義した関数やデータ型を変更するために逆戻りすることは決してありませんでした。

第三回では、高階関数と型クラスの助けを借りてインクリメンタル開発を行います。目標はゲームの進行を予測する方法を与えることです。