# オブジェクト生成
前項で、プレイヤーからの入力を取得する処理を実装しました。これでシューティングゲームに必要な、「自機がショットを撃つ」「自機が移動する」を実装することができるようになりました。本稿では、まず「自機がショットを撃つ」を実装したいと思います。

## ObjectBaseクラスの作成
今まで画面上のオブジェクトには、`Chara` クラスしかありませんでした。`Chara` クラスに加えて追加で `Shot` クラスを実装したいと思います。
`Chara` クラスには、自機固有の処理とオブジェクトとして共通の処理が実装されていました。オブジェクトで共通している処理は、`ObjectBase` クラスに切り出して、
`Chara` クラスと、`Shot` クラスは、`ObjectBase` クラスを継承するようにしてしまいましょう。

`src/util.js`
```
'use strict';
var Util = function (){};
Util.inherit = function( child, parent ) {
	var getPrototype = function(p) {
		if(Object.create) return Object.create(p);

		var F = function() {};
		F.prototype = p;
		return new F();
	};
	child.prototype = getPrototype(parent.prototype);
	child.prototype.constructor = child;
};
module.exports = Util;
```

`src/object/base.js`
```
var id = 0;

var ObjectBase = function (scene) {
	this.scene = scene;
	this.game = game;

	this._id = ++id;

	this.x = 0;
	this.y = 0;

	// 経過フレーム数
	this.frame_count = 0;

	// 現在表示するスプライト
	this.current_sprite_index = 0;
};
ObjectBase.prototype.id = function () {
	return this._id;
};
// 更新
ObjectBase.prototype.update = function () {
	this.frame_count++;

	// animation sprite
	if(this.frame_count % this.spriteAnimationSpan() === 0) {
		this.current_sprite_index++;
		if(this.current_sprite_index >= this.spriteIndices().length) {
			this.current_sprite_index = 0;
		}
	}

};
// 描画
ObjectBase.prototype.draw = function () {
	var image = this.core.image_loader.getImage(this.spriteName());

	var ctx = this.core.ctx;

	ctx.save();

	// set position
	ctx.translate(this.x, this.y);

	var sprite_width  = this.spriteWidth();
	var sprite_height = this.spriteHeight();

	ctx.drawImage(
		image,
		// sprite position
		sprite_width * this.spriteIndexX(), sprite_height * this.spriteIndexY(),
		// sprite size to get
		sprite_width,                       sprite_height,
		// adjust left x, up y because of x and y indicate sprite center.
		-sprite_width/2,                    -sprite_height/2,
		// sprite size to show
		sprite_width,                       sprite_height
	);
	ctx.restore();
};

ObjectBase.prototype.spriteName = function(){
	throw new Error("must be implemented");
};
ObjectBase.prototype.spriteIndexX = function(){
	return this.spriteIndices()[this.current_sprite_index].x;
};
ObjectBase.prototype.spriteIndexY = function(){
	return this.spriteIndices()[this.current_sprite_index].y;
};
ObjectBase.prototype.spriteAnimationSpan = function(){
	throw new Error("must be implemented");
};
ObjectBase.prototype.spriteIndices = function(){
	throw new Error("must be implemented");
};
ObjectBase.prototype.spriteWidth = function(){
	throw new Error("must be implemented");
};
ObjectBase.prototype.spriteHeight = function(){
	throw new Error("must be implemented");
};
module.exports = ObjectBase;
```

`src/object/chara.js`
```
var Util = require("../util");
var ObjectBase = require("./base");
var Chara = function (scene) {
	ObjectBase.apply(this, arguments);
};
Util.inherit(Chara, ObjectBase);

Chara.prototype.spriteName = function(){
	return "chara";
};
Chara.prototype.spriteAnimationSpan = function(){
	return 10;
};
Chara.prototype.spriteIndices = function(){
	return [{x: 0, y: 0},{x: 1, y: 0}];
};
Chara.prototype.spriteWidth = function(){
	return 64;
};
Chara.prototype.spriteHeight = function(){
	return 64;
};
module.exports = Chara;
```

`Util` クラスは、ゲームに必要なユーティリティ関数をまとめた静的なクラスです。
JavaScript はプロトタイプベースの言語であり、継承を行うには少し特殊な実装が必要です。
継承を行う `inherit` 関数を `Util` クラスに作ることにします。
`Util.inherit(ChildClass, ParentClass)` で、親クラスのプロパティやメソッドを子クラスに継承することができます。

`ObjectBase` クラスは、`Chara` クラスや `Shot` クラスが継承する基底クラスです。
前回の描画の項目などで記載したスプライトアニメーションなどの処理はここにまとめておきます。
(正確に言うと、スプライトを使わなかったり、スプライトのアニメーションをしないオブジェクトも今後増えていくので、
`SpriteAnimationObjectBase` 等のような名前が適切かと思いますが、
今回作るゲームの要件では、そこまで必要ではないので、このままで行こうと思います)

`Chara` クラスは、`ObjectBase` クラスを継承します。今まで `Chara` クラスに実装したほとんどの機能が、
`ObjectBase` クラスに移動したので、 `Chara` クラス自身は、
スプライトの設定などの `Chara` 特有の設定を記載するだけのコードになりました。

## ショット
それでは、`Shot` クラスを実装したいと思います。

**スプライト画像**  
`public/image/shot.png`

弾のスプライト画像は、本書では用意できなかったので、
読者の皆様の方でご用意頂けましたらと思います。

フリー素材としては、例えば東方弾幕風向けの素材として、以下があったりします。  
http://danmakufu.wiki.fc2.com/wiki/%E7%B4%A0%E6%9D%90%E3%83%AA%E3%83%B3%E3%82%AF  
弾画像 弾のズレが最も少ない画像。imgフォルダに入れるだけで使えるタイプ。

`src/object/shot.js`
```
var Util = require("../util");
var ObjectBase = require("./base");
var Shot = function (scene, x, y) {
	ObjectBase.apply(this, arguments);

	this.x = x;
	this.y = y;
};

Util.inherit(Shot, ObjectBase);

Shot.prototype.spriteName = function(){
	return "shot";
};
Shot.prototype.spriteAnimationSpan = function(){
	return 0;
};
Shot.prototype.spriteIndices = function(){
	return [{x: 0, y: 0}];
};
Shot.prototype.spriteWidth = function(){
	return 16;
};
Shot.prototype.spriteHeight = function(){
	return 16;
};
module.exports = Shot;
```

スプライトアニメーションを使用しないため、スプライトを1つだけ指定しています。それ以外については、`Chara` クラスと同様に、基本的な実装は、`ObjectBase` クラスを継承し、`Shot` クラス特有の情報だけ定数で定義しています。

それでは、先ほどの `Chara` クラスに、プレイヤーからの入力を受け付けて、ショットを生成する処理を追加したいと思います。

`src/object/chara.js`
```
Chara.prototype.update = function () {
	BaseObject.run.apply(this, arguments); // 親クラスの run を実行

	// Zが押下されていればショット生成
	if(this.core.isKeyDown(Constant.BUTTON_Z)) {
		this.scene.addObject(new Shot(this, this.x, this.y));
	}
};
```

先述の入力の項目で述べたプレイヤーからの入力を取得する方法に従い、
`Chara` クラス内で、プレイヤーの Z ボタンの押下を取得して、
`Shot` インスタンスを生成し、ステージに追加しています。

これで、Zボタンを押下することで、キャラが弾を撃つようになりました。
しかしショットの動きをまだ追加していないため、ステージ上の自機の位置にショットが
描画されたまま動かないと思います。これは、事項の移動の項目で修正するので、一旦敵の生成について
解説していきたいと思います。

## 敵

敵クラスの作成も行います。これは今までに解説してきたことだけで実装できます。

**スプライト画像**  
`public/image/enemy.png`

敵のスプライト画像は、本書では用意できなかったので、
読者の皆様の方でご用意頂けましたらと思います。

フリー素材としては、例えば東方弾幕風向けの素材として、以下があったりします。

http://danmakufu.wiki.fc2.com/wiki/%E7%B4%A0%E6%9D%90%E3%83%AA%E3%83%B3%E3%82%AF  
KMAPさんの素材詰め合わせ

`src/object/enemy.js`
```
var Util = require("../util");
var ObjectBase = require("./base");
var Enemy = function (scene, x, y) {
	ObjectBase.apply(this, arguments);

	this.x = x;
	this.y = y;
};
Util.inherit(Enemy, ObjectBase);
Enemy.prototype.spriteName = function(){
	return "enemy";
};
Enemy.prototype.spriteAnimationSpan = function(){
	return 0;
};
Enemy.prototype.spriteIndices = function(){
	return [{x: 0, y: 0}, {x: 0, y: 1}, {x:0, y:2}];
};
Enemy.prototype.spriteWidth = function(){
	return 16;
};
Enemy.prototype.spriteHeight = function(){
	return 16;
};
module.exports = Enemy;
```

敵を表すクラスである `Enemy` クラスです。これまでの `Chara` クラスや、`Shot` クラスと同様に、
`ObjectBase` クラスを継承して、実装しています。
この敵の出現処理ですが、次の項目の敵マスタークラスに委ねることにします。

## 敵マスター
敵の出現については、いつどこから出現するのかのロジックを考えて実装しなくてはなりません。

こうしたロジックについて、敵マスターという概念を導入し、敵マスターに出現ロジックをまとめると便利です。

`src/logic/master.js`
```
var Master = function (scene) {
	this.scene = scene;

	this.frame_count = 0;
};
Master.prototype.update = function () {
	this.frame_count++;

	if(this.frame_count % 100 === 0) {
		var x = Math.floor(Math.random() * this.core.width); // x軸はランダム
		var y = 0; // 画面上部から
		this.scene.addObject(new Enemy(this.scene, x, y));
	}
};
```

敵マスターは、概念上存在するだけで、ゲームでは実際に見えないしプレイヤーからも認知されない存在です。
こうした敵を出現させる存在を、透明のオブジェクトとして `Object` クラスで実装し、扱うゲームエンジンも存在します。
その方が、既存のオブジェクト管理の仕組みに乗っかって
実装できるので楽なのですが、今回は、オブジェクトの仕組みに乗っからずに実装したいと思います。

出現ロジックは極めてシンプルです。100フレームごとに1つ、敵を出現させます。
出現位置は、画面上部のどこからかランダムです。

`src/scene/stage.js`
```
var Master = require("../logic/master");
var StageScene = function (game) {
	/* ~ 以上省略 ~ */
	this.master = new Master(this);
};
// 更新
StageScene.prototype.update = function () {
	/* ~ 以上省略 ~ */

	this.master.update(); // 敵の出現
};
```

最後に、`Stage` クラスにて、`Master` クラスのインスタンスを追加し、
フレームごとに `Master` の `update` メソッドを実行するように修正しました。
