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
``


## vector
## aimed

