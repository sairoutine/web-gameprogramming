# 移動

前回までの項で、画面内に敵とショットのオブジェクトを生成するところまで書きました。
しかし、これらの自機・敵・自機のショットはまだ画面上で動いていません。

本稿では、画面内に存在するオブジェクトの移動について書きたいと思います。

## キャラ移動
入力の項で、プレイヤーのキーボードあるいはゲームパッドから情報を取得できるようになりました。
それでは、プレイヤーの入力を元に、キャラの移動を実装しようと思います。

`src/object/chara.js`
```
var Constant = require("../constant");

// キャラの移動速度
var SPEED = 2;
Character.prototype.run = function(){
	/* ~ 以上省略 ~ */

	// 自機移動
	if(this.core.input.isKeyDown(Constant.BUTTON_LEFT)) {
		this.x -= SPEED;
	}
	if(this.core.input.isKeyDown(Constant.BUTTON_RIGHT)) {
		this.x += SPEED;
	}
	if(this.core.input.isKeyDown(Constant.BUTTON_DOWN)) {
		this.y += SPEED;
	}
	if(this.core.input.isKeyDown(Constant.BUTTON_UP)) {
		this.y -= SPEED;
	}

	// 画面外に出させない
	this.forbidOutOfStage();
};
// 画面外に出させない
Character.prototype.forbidOutOfStage = function(){
	if(this.x < 0) {
		this.x = 0;
	}
	if(this.x > this.scene.width) {
		this.x = this.core.width;
	}
	if(this.y < 0) {
		this.y = 0;
	}
	if(this.y > this.core.height) {
		this.y = this.core.height;
	}
};
```
`Chara` クラスを少し修正しました。`run` メソッド内で、プレイヤーの入力を受け取って、自機の(x, y)座標を移動させています。
また、`forbidOutOfStage` メソッドにより自機が画面の外に出てしまわないように、
画面外に出てしまったら、画面内に座標を戻すように修正しています。

## ベクトル移動
前の実装でキャラの移動について実装しました。(x, y)の加算・減算だけで実現しましたが、
この方法は、斜め方向の移動等、細かい移動をするには不便です。
オブジェクトを移動させたい際に、「速度」「角度」を指定することで移動できるようになると便利です。

キャラの移動について実装したので、次は自機が撃つショットの移動を実装しましょう。
自機の撃つショットは、前方向に進んでいきます。

`src/util.js`
```
Util.thetaToRadian = function(theta) {
	return theta * Math.PI / 180;
};
Util.calcMoveX = function(speed, theta) {
	return speed * Math.cos(Util.thetaToRadian(theta));
};
Util.calcMoveY = function(speed, theta) {
	return speed * Math.cos(Util.thetaToRadian(theta));
};
```

`src/object/base.js`
```
var Util = require("./util");
var ObjectBase = function (scene) {
	/* ~以上省略 ~ */

	this.speed = 0;
	this.theta = 0;
};
ObjectBase.prototype.update = function () {
	/* ~以上省略 ~ */

	this.move();
};
ObjectBase.prototype.move = function() {
	if (this.speed === 0) return;

	var x = Util.calcMoveX(this.speed, this.theta);
	var y = Util.calcMoveY(this.speed, this.theta);
	this.x += x;
	this.y += y;
};
ObjectBase.prototype.setMove = function(speed, theta) {
	this.speed = speed;
	this.theta = theta;
};
```

`ObjectBase` クラスを少し修正しました。`ObjectBase` クラスを継承したオブジェクトは、
`setMove` メソッドに、`speed` と `theta` を指定することで、後はプログラムが自動でそれに応じた移動をしてくれます。
`theta` はどの方向に向かうかの角度、`speed` は1フレームにどのくらい進むかの速度です。

`theta` はシータと言います。記号では θ と記述します。0 ~ 360 の値を取り、
右が 0、下が、90、左が180, 上が 270 となります。(360 は 0 と同じく右です。)

`speed * cos(θ)` を計算すると、θ 方向に `speed` の分だけ移動する際に、x 座標はどのくらい進むのかわかります。
同じく、`speed * sin(θ)` を計算すると、θ 方向に `speed` の分だけ移動する際に、y 座標はどのくらい進むのかわかります。

なお、JavaScript の `sin` `cos` 関数は、θ ではなく、ラジアンを渡さないといけないので、
`Util.thetaToRadian` 関数により、θ からラジアンに変換しています。

それでは、`setMove` を使って、自機のショットに移動を設定してみたいと思います。

`src/object/chara.js`
```
// ショットの移動速度
var SHOT_SPEED = 3;
var SHOT_THETA = 270; // 画面前方
Chara.prototype.update = function () {
	BaseObject.run.apply(this, arguments); // 親クラスの run を実行

	// Zが押下されていればショット生成
	if(this.core.isKeyDown(Constant.BUTTON_Z)) {
		var shot = new Shot(this, this.x, this.y);
		shot.setMove(SHOT_SPEED, SHOT_THETA);

		this.scene.addObject(shot);
	}
};
```

`setMove` 関数を使用し、`Chara` クラス内で、ショットを生成する際に、ショットの角度と速度を設定しています。
これで、Zボタンを押すと、自機から弾が発射され、画面前方に進んでいくようになったと思います。

## 自機狙い

自機狙いなど、オブジェクトがある特定の位置に向かって移動する設定をするにはどうすればいいでしょうか。
移動自体は、`setMove` に `speed` と `theta` を渡してあげればいいのですが、オブジェクトの位置と、移動先の位置から、
`speed` と `theta` を計算する必要があります。

`src/util.js`
```
/* ~ 以下を追加 ~ */
Util.radianToTheta = function(radian) {
	return (radian * 180 / Math.PI) | 0;
};
```

`src/object/base.js`
```
/* ~ 以下を追加 ~ */
ObjectBase.prototype.setAimTo = function(x, y) {
	var ax = x - this.x;
	var ay = y - this.y;

	var theta = Util.radianToTheta(Math.atan2(ay, ax));
	return theta;
};
```

JavaScript には `atan2` 関数があるので、これを使うことで、2つの(x, y)座標から、
どの方向に向かうかの角度を取得することができます。
ただし、`atan2` から返ってきた角度は、ラジアンなので、θに変換する必要があります。

これで例えば、敵を自機に向かって移動させたい場合は、

`src/object/enemy.js`
```
Enemy.prototype.update = function () {
	/* ~以上省略 ~ */

	if (this.frame_count % 60 === 0) {
		this.setAimTo(this.scene.chara.x, this.scene.chara.y);
	}
};
```

とすることで、自機に向かって移動してくる敵にすることができます。

毎フレーム、自機の移動先を計算すると処理に時間がかかるので、60フレームごとに
自機の位置を取得して、向かう先を修正しています。

