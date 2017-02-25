# 移動

本稿では、画面内のオブジェクトの移動について書きたいと思います。

## キャラ移動
入力の項目より、プレイヤーのキーボードあるいはゲームパッドから情報を取得できるようになりました。
それでは、プレイヤーの入力を元に、キャラの移動を実装しようと思います。
`src/js/chara.js`
```
var Constant = require("../constant");
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

run メソッド内で、プレイヤーの入力を受け取って、自機のx, y座標を移動させています。
また、forbidOutOfStage メソッドにより、自機が画面の外に出てしまわないように、
画面外に出てしまったら、画面内に座標を戻すように修正しています。

## vector
移動ですが、x, y 座標の加算／減算だけだと、斜め方向の移動等、細かい移動をするには不便です。
「速度」「角度」を指定することで移動できると便利です。

キャラの移動について実装したので、次は自機が撃つ弾の移動を実装しましょう。
自機の撃つ弾は、前方向に進んでいきます。

`src/js/util.js`
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

`src/js/object/base.js`
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
}
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

theta はどの方向に向かうかの角度、speed は1フレームにどのくらい進むかの速度です。


theta はシータと言います。記号では θ と記述します。0 ~ 360 の値を取り、
右が 0、下が、90、左が180, 上が 270 となります。(360 は 0 と同じく右です。)

# TODO:
図(シータの)

speed * cos(θ) を計算すると、θ方向に speed 移動する際に、x 座標はどのくらい進むのかわかります。
同じく、
speed * sin(θ) を計算すると、θ方向に speed 移動する際に、y 座標はどのくらい進むのかわかります。

# TODO:
sin cos 関係の図

`src/js/object/chara.js`
```
var SHOT_SPEED = 3;
var SHOT_THETA = 270; // 前方
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

上記のように、キャラクラス内で、ショットを生成する際に、ショットの角度と速度を設定してあげます。
これで、Zボタンを押すと、自機から弾を発射するようになったと思います。


## aimed

