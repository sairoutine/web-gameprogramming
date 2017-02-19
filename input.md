# 入力

本稿では、プレイヤーがキーボードあるいはゲームパッド(ゲーム用のコントローラーのこと)を受け取る方法について解説します。

## キーボード入力
まず、ユーザーのキーボード入力を受け取るクラスを作りたいと思います。

`src/constant.js`
```
var Constant = {
	BUTTON_LEFT:  0x01, // 0b00000001
	BUTTON_UP:    0x02, // 0b00000010
	BUTTON_RIGHT: 0x04, // 0b00000100
	BUTTON_DOWN:  0x08, // 0b00001000
	BUTTON_Z:     0x10, // 0b00010000
	BUTTON_X:     0x20, // 0b00100000
	BUTTON_SHIFT: 0x40, // 0b01000000
	BUTTON_SPACE: 0x80, // 0b10000000
};
module.exports = Constant;
```

こちらはキー入力を表す定数です。後述しますが、キーが入力されたかどうかの情報は、ビットで管理することにします。
これは以下のメリットのためです。

・ビット演算子の方が通常の算術より高速
・変数一つで、複数のキー押下を管理することができる。

ES5では、2進数による値表記ができないため、16進数で定数を定義しています。
2進数にすると、どういう値になるかは、コメントに記載しました。

ご覧の通り、該当のキーに対応するビットが立っていれば、押下されていると判定します。

`src/input.js`
```
var Consntant = require("./constant");
var Input = function() {
	// キー押下フラグ
	this.keyflag = 0x0;

	// 一つ前のフレームで押下されたキー
	this.before_keyflag = 0x0;
};
// キー押下
Input.prototype.handleKeyDown = function(e){
	this.keyflag |= this._keyCodeToBitCode(e.keyCode);
	e.preventDefault();
};
// キー押下解除
Input.prototype.handleKeyUp = function(e){
	this.keyflag &= ~this._keyCodeToBitCode(e.keyCode);
	e.preventDefault();
};
// 指定のキーが押下状態か確認する
Input.prototype.isKeyDown = function(flag){
	return this.keyflag & flag;
};
// 指定のキーが押下されたか確認する
Input.prototype.isKeyPush = function(flag){
	// 1フレーム前に押下されておらず、現フレームで押下されてるなら true
	return !(this.before_keyflag & flag) && this.keyflag & flag;
};
// キーコードをBitに変換
Input.prototype._keyCodeToBitCode = function(keyCode){
	var flag;
	switch(keyCode) {
		case 16: // shift
			flag = Constant.BUTTON_SHIFT;
			break;
		case 32: // space
			flag = Constant.BUTTON_SPACE;
			break;
		case 37: // left
			flag = Constant.BUTTON_LEFT;
			break;
		case 38: // up
			flag = Constant.BUTTON_UP;
			break;
		case 39: // right
			flag = Constant.BUTTON_RIGHT;
			break;
		case 40: // down
			flag = Constant.BUTTON_DOWN;
			break;
		case 88: // x
			flag = Constant.BUTTON_X;
			break;
		case 90: // z
			flag = Constant.BUTTON_Z;
			break;
	}
	return flag;
};
Input.prototype.bindKey = function(){
	var self = this;
	window.onkeydown = function(e) { self.handleKeyDown(e); };
	window.onkeyup   = function(e) { self.handleKeyUp(e); };
};
Input.prototype.saveBeforeKey = function(){
	this.before_keyflag = this.keyflag;
	this.keyflag = 0x0;
}
```

`src/game.js`
```
var Game = function (canvas) {
	this.input = new Input();
	this.input.bindKey();

	/* ~ 以下省略 ~ */
};
Game.prototype.run = function() {
	/* ~ 以上省略 ~ */

	// 押下されたキーを保存しておく
	this.input.saveBeforeKey();
};
```

それでは少しずつ解説していきます。

```
var Input = function() {
	// キー押下フラグ
	this.keyflag = 0x0;

	// 一つ前のフレームで押下されたキー
	this.before_keyflag = 0x0;
};
```

Input クラスのコンストラクタです。プロパティとして、押下されたキーを保存する `keyflag` と、1フレーム前に押下されたキーを保存する `before_keyflag` を持ちます。先述の通り、`keyflag` は 8bit の値として使い、各bit 毎に bit が立っているかどうかで、どのキーが押下されているかを判定します。

`before_keyflag` には、1フレーム前に押下されたキーを保存します。なぜこの変数が必要なのでしょうか。それは、キーの押下判定を、JavaScript で取得するには、onkeyup と onkeydown イベントハンドラを利用するのですが、onkeydown は、キーが押下されていれば、実行されます。ボタンが押しっぱなしかどうかを判定する分には、便利なのですが、ボタンが最初に押下された時を取得する場合、現在のフレームでキーが押下されたことを確認し、1フレーム前にキーが押下されていないことを確認しないといけないからです。

```
Input.prototype.bindKey = function(){
	var self = this;
	window.onkeydown = function(e) { self.handleKeyDown(e); };
	window.onkeyup   = function(e) { self.handleKeyUp(e); };
};
Input.prototype.saveBeforeKey = function(){
	this.before_keyflag = this.keyflag;
	this.keyflag = 0x0;
}
```

bindKey メソッドはゲームが起動した時に最初に一度だけ呼び出します。ブラウザのキー入力を、取得し、
`KeyboardEvent` オブジェクトを、`handleKeyDown` 及び `handleKeyUp` メソッドに渡す設定を行います。

`saveBeforeKey` メソッドは、フレーム処理の最後に毎回実行します。
入力されたキーの履歴を保存します。


```
// キーコードをBitに変換
Input.prototype._keyCodeToBitCode = function(keyCode){
	var flag;
	switch(keyCode) {
		case 16: // shift
			flag = Constant.BUTTON_SHIFT;
			break;
		case 32: // space
			flag = Constant.BUTTON_SPACE;
			break;
		case 37: // left
			flag = Constant.BUTTON_LEFT;
			break;
		case 38: // up
			flag = Constant.BUTTON_UP;
			break;
		case 39: // right
			flag = Constant.BUTTON_RIGHT;
			break;
		case 40: // down
			flag = Constant.BUTTON_DOWN;
			break;
		case 88: // x
			flag = Constant.BUTTON_X;
			break;
		case 90: // z
			flag = Constant.BUTTON_Z;
			break;
	}
	return flag;
};
// キー押下
Input.prototype.handleKeyDown = function(e){
	this.keyflag |= this._keyCodeToBitCode(e.keyCode);
	e.preventDefault();
};
// キー押下解除
Input.prototype.handleKeyUp = function(e){
	this.keyflag &= ~this._keyCodeToBitCode(e.keyCode);
	e.preventDefault();
};
```

`_keyCodeToBitCode` 関数は、onkeyup もしくは onkeydown イベントが発生した際に渡される
`KeyboardEvent` オブジェクトの `keyCode` です。keyCode のどの値が、キーボードの
どのキーに紐づくかは、MDN(https://developer.mozilla.org/ja/docs/Web/API/KeyboardEvent/keyCode)などを参照してください。

handleKeyDown は、キーが押下された際に実行される関数です。内部では、キーコードを Bit に変換し、
OR 演算子を使って、 keyflag に保存しています。ビット演算子は、Webエンジニアにとって慣れ親しまないと思うので、
詳しく解説します。

Zボタンが押されると、handleKeyDown は以下を実行します。

```
	//BUTTON_Z => 0b00010000
	this.key_flag = 0b00000000 | 0b00010000;
```

各bit それぞれについて、どちらか片方の値が 1 であれば、そのbitは1となります。
つまり、key_flag は `0b00010000` となります。

Zボタンと同時にXボタンも押されていればどうなるでしょうか

```
	//BUTTON_X => 0b00100000
	this.key_flag = 0b00010000 | 0b00100000;
```

既にZボタンが押されているので、`0b00010000` に対して OR 演算を行います。
各ビットに対して、片方の値が 1 になります。よって、既に押下されている Z ボタンの状態は残ったまま、
Xボタンも押された状態になります。`0b00110000` となります。


handleKeyUp は、キーが押下された際に実行される関数です。内部では、キーコードを Bit に変換し、
OR 演算子を使って、 keyflag に保存しています。ビット演算子は、Webエンジニアにとって慣れ親しまないと思うので、
詳しく解説します。

Zボタンが押されると、handleKeyDown は以下を実行します。

```
	//BUTTON_Z => 0b00010000
	this.key_flag = 0b00000000 | 0b00010000;
```

各bit それぞれについて、どちらか片方の値が 1 であれば、そのbitは1となります。
つまり、key_flag は `0b00010000` となります。

Zボタンと同時にXボタンも押されていればどうなるでしょうか

```
	//BUTTON_X => 0b00100000
	this.key_flag = 0b00010000 | 0b00100000;
```

既にZボタンが押されているので、`0b00010000` に対して OR 演算を行います。
各ビットに対して、片方の値が 1 になります。よって、既に押下されている Z ボタンの状態は残ったまま、
Xボタンも押された状態になります。`0b00110000` となります。




















```
// 指定のキーが押下状態か確認する
Input.prototype.isKeyDown = function(flag){
	return this.keyflag & flag;
};
// 指定のキーが押下されたか確認する
Input.prototype.isKeyPush = function(flag){
	// 1フレーム前に押下されておらず、現フレームで押下されてるなら true
	return !(this.before_keyflag & flag) && this.keyflag & flag;
};
```














`src/game.js`
```
var Game = function (canvas) {
	this.input = new Input();
	this.input.bindKey();

	/* ~ 以下省略 ~ */
};
Game.prototype.run = function() {
	/* ~ 以上省略 ~ */

	// 押下されたキーを保存しておく
	this.input.saveBeforeKey();
};
```

最後に、キー入力を受け取るために、Game クラスを少し修正します。

## それ以外のキー情報の保存方法

キー情報の保存方法は、いくつか種類があります。前述したのは、
「前の押下情報だけを保存しておく」方法でした。これは、キーが押下されているか、あるいはキーが押下されたかを
判定するだけのゲームであればこの方法で充分です。

他の保存方法には、「コマンド履歴を保存する」「押下時間も保存する」方法などがあります。
「コマンド履歴を保存する」方法は、格闘ゲームなどの、プレイヤーが入力したコマンド履歴を参照する場合に用いられます。
「押下時間も保存する」タイプは、例えばシューティングゲームのチャージショットなどを利用する場合に用いられます。

# TODO: 
コード例

## ゲームパッド入力

	handleGamePad: function() {
		if(!this.is_connect_gamepad) return;

		var pads = navigator.getGamepads();
		var pad = pads[0]; // 1Pコン

		if(!pad) return;

		this.keyflag = 0x00;
		this.keyflag |= pad.buttons[1].pressed ? constant.BUTTON_Z:      0x00;// A
		this.keyflag |= pad.buttons[0].pressed ? constant.BUTTON_X:      0x00;// B
		this.keyflag |= pad.buttons[2].pressed ? constant.BUTTON_SELECT: 0x00;// SELECT
		this.keyflag |= pad.buttons[3].pressed ? constant.BUTTON_START:  0x00;// START
		this.keyflag |= pad.buttons[4].pressed ? constant.BUTTON_SHIFT:  0x00;// SHIFT
		this.keyflag |= pad.buttons[5].pressed ? constant.BUTTON_SHIFT:  0x00;// SHIFT
		this.keyflag |= pad.buttons[6].pressed ? constant.BUTTON_SPACE:  0x00;// SPACE
		//this.keyflag |= pad.buttons[8].pressed ? 0x04 : 0x00;// SELECT
		//this.keyflag |= pad.buttons[9].pressed ? 0x08 : 0x00;// START

		this.keyflag |= pad.axes[1] < -0.5 ? constant.BUTTON_UP:         0x00;// UP
		this.keyflag |= pad.axes[1] >  0.5 ? constant.BUTTON_DOWN:       0x00;// DOWN
		this.keyflag |= pad.axes[0] < -0.5 ? constant.BUTTON_LEFT:       0x00;// LEFT
		this.keyflag |= pad.axes[0] >  0.5 ? constant.BUTTON_RIGHT:      0x00;// RIGHT
	},

