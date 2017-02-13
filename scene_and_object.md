# シーンとオブジェクト
## 概要
ゲームのプログラムを書いていく上で重要な概念として、シーン(Scene)とオブジェクト(Object)があります。オブジェクトはアクター(Actor)と呼ばれる場合があるかもしれません。また、シーンもオブジェクトの1種として扱うゲームエンジンもあります。

シーンは、Webサイトでいうところのページです。画面1つを表します。例えば、タイトルシーン、ステージシーン、エンディングシーンなどがあります。シーンは遷移します。タイトルシーンの次は、ステージ選択シーンに変わるでしょうし、ステージシーンでゲームオーバーになれば、ゲームオーバーシーンに変わります。

オブジェクトは Web サイトでいうところの DOM です。例えば、敵オブジェクト、自機オブジェクト、弾オブジェクト、文字オブジェクトなどがあります。シーン上に存在する全ての要素を、オブジェクトとして扱います。オブジェクトには親子関係があります。シューティングゲームで言えば、自機とオプションは、親子関係になるでしょう。

シーンとオブジェクトは、シューティングゲームでもアクションゲームでも通用する概念です。2Dでも3Dでも通用する概念です。
## シーン
それでは、実際にシーンを作ってみましょう。まずはステージシーンを作ろうと思います。
`src/scene/stage.js`
```
var StageScene = function (game) {
	this.game = game;

	// 経過フレーム数
	this.frame_count = 0;
};
// 更新
StageScene.prototype.update = function () {
	this.frame_count++;
};
// 描画
StageScene.prototype.draw = function () {

};
module.exports = StageScene;
```

StageScene クラスを作りました。StageScene クラスはインスタンスメソッドとして、
update と draw メソッドを持っています。

update と draw メソッドは、1秒間に 60回呼び出すこととします。
update メソッドはゲームの状態の更新を担当し、draw はその後に状態を元に描画することを担当することにします。

update 関数の中では、経過フレーム数を数えています。ゲームの種類にもよりますが、
シューティングゲームなどでは、敵の登場タイミングを計る際などに使用するので、
経過フレーム数を数えておくと、他の実装でも使うことができます。

コンストラクタの引数に `game` 引数を取っています。`game` は Game クラスのインスタンスを渡すことにします。


## シーン切り替え
シーンを Game クラス側から呼び出したいと思います。
`src/game.js`
```
var StageScene = require("scene/stage");
var Game = function (canvas) {
	this.ctx = canvas.getContext('2d'); // Canvas への描画は、ctx プロパティを通して行うこととする。
	// 画面サイズ
	this.height = canvas.height;
	this.width = canvas.width;

	this.scenes = {}; // シーン一覧
	this.next_scene = null;
	this.current_scene = null;
	this.prev_scene = null;

	this.addScene("stage", new StageScene(this)); // シーンを追加
	this.changeScene("stage"); // 最初のシーンに切り替え
};
Game.prototype.run = function () {
	this.toNextSceneIfExists(); // 次のシーンが前フレームにて予約されていればそちらに切り替え

	this.request_id = requestAnimationFrame(this.run.bind(this));
};

Game.prototype.addScene = function (name, scene) {
	this.scenes[name] = scene;
};
Game.prototype.changeScene = function (name) {
	this.next_scene = name;
};
Game.prototype.toNextSceneIfExists = function () {
	if(!this.next_scene) return;

	this.current_scene = this.next_scene;
	this.next_scene = null;
};
Game.prototype.changePrevScene = function () {
	if(!this.prev_scene) return;

	this.current_scene = this.prev_scene;
	this.prev_scene = null;
};
```

Game クラスのコンストラクタ及び `run` 関数を修正しております。また、`addScene`, `changeScene`, `toNextSceneIfExists`, `changePrevScene` メソッドを追加しております。一つずつ解説したいと思います。

```
var Game = function (canvas) {
	this.ctx = canvas.getContext('2d'); // Canvas への描画は、ctx プロパティを通して行うこととする。
	// 画面サイズ
	this.height = canvas.height;
	this.width = canvas.width;

	this.scenes = {}; // シーン一覧
	this.next_scene = null;
	this.current_scene = null;
	this.prev_scene = null;

	this.addScene("stage", new StageScene(this)); // シーンを追加
	this.changeScene("stage"); // 最初のシーンに切り替え
};
```
Game クラスのコンストラクタを変更してます。大きな変更は2点です。

まずシーン管理用に `scenes`, `next_scene`, `current_scene`, `prev_scene` といったプロパティの追加をしています。

`scenes` には、ゲームで使用するシーン一覧を追加します。後述しますが、追加は `addScene` メソッドを使って行います。
基本的に、`addScene` は、動的にシーンを生成して追加する要件がないかぎり、
ゲームを初期化する際に、全てのシーンを追加することとなるかと思います。










と、`addScene`, `changeScene` メソッドを追加しております。

```
Game.prototype.run = function () {
	this.toNextSceneIfExists(); // 次のシーンが前フレームにて予約されていればそちらに切り替え

	this.request_id = requestAnimationFrame(this.run.bind(this));
};
```

```
Game.prototype.addScene = function (name, scene) {
	this.scenes[name] = scene;
};
```


```
Game.prototype.changeScene = function (name) {
	this.next_scene = name;
};
```

```
Game.prototype.toNextSceneIfExists = function () {
	if(!this.next_scene) return;

	this.current_scene = this.next_scene;
	this.next_scene = null;
};
```



```
Game.prototype.changePrevScene = function () {
	if(!this.prev_scene) return;

	this.current_scene = this.prev_scene;
	this.prev_scene = null;
};
```











なお、これらのシーン管理は、いわゆる Game クラスにさせるのではなく、`SceneManager` のような専用のクラスを
用意して、そこに `addScene`, `changeScene` といったメソッドをまとめる場合も多いです。

## オブジェクト




