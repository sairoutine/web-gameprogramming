# シーンとオブジェクト
## 概要
ゲームのプログラムを書いていく上で重要な概念として、シーン(Scene)とオブジェクト(Object)があります。オブジェクトはアクター(Actor)と呼ばれる場合があるかもしれません。また、シーンもオブジェクトの1種として扱うゲームエンジンもあります。

シーンは、Webサイトでいうところのページです。画面1つを表します。例えば、タイトルシーン、ステージシーン、エンディングシーンなどがあります。シーンは遷移します。タイトルシーンの次は、ステージ選択シーンに変わるでしょうし、ステージシーンでゲームオーバーになれば、ゲームオーバーシーンに変わります。

オブジェクトは Web サイトでいうところの DOM です。例えばシューティングゲームであれば、敵オブジェクト、自機オブジェクト、弾オブジェクト、文字オブジェクトなどがあります。シーンとオブジェクトの仕組みを採用したゲームエンジンでは、シーン上に存在する全ての要素を、オブジェクトとして扱うことになります。そしてオブジェクトには親子関係があります。シューティングゲームで言えば、自機と自機オプションは、親子関係になるでしょう。

シーンとオブジェクトは、シューティングゲームでもアクションゲームでも通用する概念です。また2Dでも3Dでも通用するゲームの設計において普遍的な概念です。

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

`StageScene` クラスを作りました。`StageScene` クラスはインスタンスメソッドとして、
`update` と `draw` メソッドを持っています。

`update` と `draw` メソッドは、1秒間に 60 回呼び出すこととします。
`update` メソッドはゲームの状態の更新を担当し、`draw` はその後に状態を元に描画することを担当することにします。

`update` 関数の中では、経過フレーム数を数えています。ゲームの種類にもよりますが、
シューティングゲームなどでは、敵の登場タイミングを計る際などに経過フレーム数を使用するので、
経過フレーム数を数えておくと、他の実装でも使うことができます。

コンストラクタの引数に `game` 引数を取っています。`game` は `Game` クラスのインスタンスを渡すことにします。

## シーン切り替え
シーンを `Game` クラス側から呼び出したいと思います。

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

	this.scenes[this.current_scene].update();
	this.scenes[this.current_scene].draw();

	this.request_id = requestAnimationFrame(this.run.bind(this));
};
Game.prototype.toNextSceneIfExists = function () {
	if(!this.next_scene) return;

	this.current_scene = this.next_scene;
	this.next_scene = null;
};

// ゲームで使用するシーンを追加
Game.prototype.addScene = function (name, scene) {
	this.scenes[name] = scene;
};
// シーン切り替え
Game.prototype.changeScene = function (name) {
	this.next_scene = name;
};
// 前のシーンに戻る
Game.prototype.changePrevScene = function () {
	if(!this.prev_scene) return;

	this.current_scene = this.prev_scene;
	this.prev_scene = null;
};
```

`Game` クラスのコンストラクタ及び `run` 関数を修正しております。また、`addScene`, `changeScene`, `toNextSceneIfExists`, `changePrevScene` メソッドを追加しております。一つずつ解説したいと思います。

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
Game クラスのコンストラクタです。大きな変更は2点です。

まずシーン管理用に `scenes`, `next_scene`, `current_scene`, `prev_scene` といったプロパティの追加をしています。

`scenes` には、ゲームで使用するシーン一覧を追加します。後述しますが、追加は `addScene` メソッドを使って行います。
基本的に、`addScene` は、動的にシーンを生成して追加する要件がないかぎり、
ゲームを初期化する際に、全てのシーンを追加することとなるかと思います。

`next_scene`, `current_scene`, `prev_scene` はそれぞれ、「次に遷移するシーン」「現在のシーン」「前のシーン」です。
後述しますが、シーンを切り替える際は、切り替えを実行して、すぐに切り替えるのではなく、次のフレームの最初にシーン切り替えを行います。そのため、切り替え先のシーンを一旦保存するための `next_scene` が必要になります。また前のシーンに戻りたい場合がゲームではあります。例えばステージシーンで、メニューを開いて、メニューを閉じたらまた元のステージに戻る場合などです。そういった用途のために、前のシーンを保存する `prev_scene` を使用します。

```
Game.prototype.run = function () {
	this.toNextSceneIfExists(); // 次のシーンが前フレームにて予約されていればそちらに切り替え

	this.scenes[this.current_scene].update();
	this.scenes[this.current_scene].draw();

	this.request_id = requestAnimationFrame(this.run.bind(this));
};
```

`run` メソッドは、requestAnimationFrame 関数により、ブラウザの描画タイミング毎(1秒に60回)呼ばれます。
`run` メソッド内にて、`toNextSceneIfExists` メソッドを呼び出しています。先述しましたが、`toNextSceneIfExists` 関数は、`next_scene` プロパティに、切り替え先のシーンが予約されていれば、そのシーンに切り替えを行います。

また、現在のシーンの `update` と `draw` メソッドを呼び出しています。`run` メソッドが1秒に60回呼ばれるので、
シーンの `update` と `draw` も1秒に60回呼ばれるわけです。

```
Game.prototype.toNextSceneIfExists = function () {
	if(!this.next_scene) return;

	this.current_scene = this.next_scene;
	this.next_scene = null;
};
```

`addScene`, `changeScene` メソッドについては以下の通りです。`name` にはシーン名を、`scene` にはシーンのインスタンスを渡します。

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

また、`changePrevScene` メソッドも追加しておきます。前のシーンに戻る関数です。

```
Game.prototype.changePrevScene = function () {
	if(!this.prev_scene) return;

	this.current_scene = this.prev_scene;
	this.prev_scene = null;
};
```

なお今回は省略しましたが、これらのシーン管理は、Game クラスに実装せず、
`SceneManager` のような専用のクラスを用意して、そこに `addScene`, `changeScene` といったメソッドをまとめる場合も多いです。

## オブジェクト
オブジェクトを作っていきましょう。自機オブジェクトを作成します。

`src/object/chara.js`
```
var Chara = function (scene) {
	this.scene = scene;
	this.game = game;

	this._id = "chara";

	this.x = 0;
	this.y = 0;

	// 経過フレーム数
	this.frame_count = 0;
};
Chara.prototype.id = function () {
	return this._id;
};
// 更新
Chara.prototype.update = function () {
	this.frame_count++;
};
// 描画
Chara.prototype.draw = function () {

};
module.exports = Chara;
```

Chara クラスを作りました。Chara クラスはインスタンスメソッドとして、
`update` と `draw` メソッドを持っています。また、パブリックなアクセサとして、`id` を持っています。

`Scene` クラスと同じように、`update` と `draw` メソッドは、1秒間に 60回呼び出すこととします。
`update` メソッドはゲームの状態の更新を担当し、`draw` はその後に状態を元に描画することを担当することにします。

`id` はゲーム内でオブジェクトを一意に識別するIDです。例えば、敵の弾などは画面上に同じ種類の弾が複数存在するので、
それらを個別に識別するIDがあると便利です。今回、キャラは画面上に1機だけなので、
`"chara"` 固定にしたいと思います。

コンストラクタの引数に `scene` 引数を取っています。`scene` は シーンクラスのインスタンスを渡すことにします。

# シーンにオブジェクトの追加
ステージシーンに、先ほど追加したキャラクラスのインスタンスを追加したいと思います。

`src/scene/stage.js`
```
var Chara = require("../object/chara");
var StageScene = function (game) {
	this.game = game;

	// シーン上のオブジェクト一覧
	this.objects = {};

	// 経過フレーム数
	this.frame_count = 0;

	this.addObject(new Chara(this));
};
StageScene.prototype.addObject = function (object) {
	this.objects[object.id()] = object;
};
// 更新
StageScene.prototype.update = function () {
	this.frame_count++;

	this.updateObjects();
};
StageScene.prototype.updateObjects = function () {
	for (var id in this.objects) {
		this.objects[id].update();
	}
};

// 描画
StageScene.prototype.draw = function () {
	this.drawObjects();

};
StageScene.prototype.drawObjects = function () {
	for (var id in this.objects) {
		this.objects[id].draw();
	}
};

module.exports = StageScene;
```

修正点をピックアップして解説していきます。

```
StageScene.prototype.addObject = function (object) {
	this.objects[object.id()] = object;
};
```

`addObject` メソッドを追加して、ステージシーンにキャラインスタンスを追加しています。
追加されたキャラインスタンスは、他のオブジェクトと一緒に、`objects` プロパティ内で管理されます。
この辺りは、`Game` クラスに、シーン管理を追加したときと同じですね。

```
// 更新
StageScene.prototype.update = function () {
	this.frame_count++;

	this.updateObjects();
};
StageScene.prototype.updateObjects = function () {
	for (var id in this.objects) {
		this.objects[id].update();
	}
};
```

シーンの `update` を実行すると、シーンが管理している各オブジェクトの `update` も実行されるようにします。

```
// 描画
StageScene.prototype.draw = function () {
	this.drawObjects();

};
StageScene.prototype.drawObjects = function () {
	for (var id in this.objects) {
		this.objects[id].draw();
	}
};
```

同様に、シーンの `draw` を実行すると、シーンが管理する各オブジェクトの `draw` も実行されるようにします。
