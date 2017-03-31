# 入力

前項までで、ゲームにシーンとオブジェクトを追加し、その中で画像を描画するところまで解説しました。
このままでは、画面は表示されていますが、プレイヤーがゲームに干渉することはできません。

本項では、プレイヤーがキーボードあるいはゲームパッド(ゲーム用のコントローラーのこと)で操作した内容をゲーム側で受け取る方法について解説します。

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
};
module.exports = Constant;
```

プレイヤーのキー入力は、定数を使ってプログラムから扱いたいと思います。後述しますが、どのキーが入力されたかどうかの情報はビットで管理することにします。
これは以下のメリットのためです。

 - ビット演算子の方が通常の算術より高速
 - 一つの変数で、複数のキー押下を管理することができる。

ES5では、2進数による値表記ができないため、16進数で定数を定義しています。
2進数にすると、どういう値になるかは、コメントに記載しました。

ご覧の通り、キー一つにつき、1bitだけ立てており、キーに対応するビットが立っていれば、押下されていると判定するようにしたいと思います。


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

`Input` クラスを作りました。また `Game` クラスから `Input` クラスを呼び出しています。
それでは少しずつ解説していきます。

```
var Input = function() {
	// キー押下フラグ
	this.keyflag = 0x0;

	// 一つ前のフレームで押下されたキー
	this.before_keyflag = 0x0;
};
```

`Input` クラスのコンストラクタです。プロパティとして、押下されたキーを保存する `keyflag` と、1フレーム前に押下されたキーを保存する `before_keyflag` を持ちます。先述の通り、`keyflag` は 8bit の値として使い、各 bit 毎に bit が立っているかどうかをチェックすることで、どのキーが押下されているかを判定します。

`before_keyflag` には、1フレーム前に押下されたキーを保存します。なぜこの変数が必要なのかを解説します。キーの押下判定を、JavaScript で取得するには、`onkeyup` と `onkeydown` イベントハンドラを利用するのですが、`onkeydown` は、キーが押下さえされていれば発火されます。この性質は、ボタンが押しっぱなしであるかどうかを判定する分には、便利なのです。一方ボタンが押下された時を取得する場合、現在のフレームでキーが押下されたことを確認し、1フレーム前にキーが押下されていないことを確認しなければいけません。そのために、1フレーム前のキー入力の情報を保存する `before_keyflag` を用意したわけです。

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

`bindKey` メソッドはゲームが起動した時に最初に一度だけ呼び出します。ブラウザのキー入力時へのフックを追加します。

`saveBeforeKey` メソッドは、フレーム処理の最後に毎回実行します。そのフレームで、プレイヤーが入力されたキーの履歴を保存します。


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

`_keyCodeToBitCode` 関数は、`onkeyup` もしくは `onkeydown` イベントが発生した際に渡される
`KeyboardEvent` オブジェクトの `keyCode` です。keyCode のどの値が、キーボードの
どのキーに紐づくかは、MDN(https://developer.mozilla.org/ja/docs/Web/API/KeyboardEvent/keyCode) などを参照してください。

`handleKeyDown` は、キーが押下された際に実行される関数です。内部では、キーコードを Bit に変換し、
OR 演算子を使って、 `keyflag` に保存しています。ビット演算子は、Webエンジニアにとって慣れ親しまないと思うので、
詳しく解説します。

Zボタンが押されると、`handleKeyDown` は以下を実行します。

```
//BUTTON_Z => 0b00010000
this.key_flag = 0b00000000 | 0b00010000;
```

OR 演算子なので、各bit それぞれについて、どちらか片方の値が 1 であれば、そのbitは1となります。
つまり、`key_flag` は `0b00010000` となります。

Zボタンと同時にXボタンも押されていればどうなるでしょうか

```
//BUTTON_X => 0b00100000
this.key_flag = 0b00010000 | 0b00100000;
```

既にZボタンが押されているので、`0b00010000` に対して OR 演算を行います。
各ビットに対して、片方の値が 1 になります。よって、既に押下されている Z ボタンの状態は残ったまま、
Xボタンも押された状態になります。`0b00110000` となります。


`handleKeyUp` は、キーの押下が解除された際に実行される関数です。
内部では、キーコードを Bit に変換し、否定演算子で、ビットを反転させてから、
AND 演算子を使って、 `keyflag` に保存しています。
また順を追って解説していきます。

Zボタンが押されるのが解除されると、`handleKeyUp` は以下を実行します。

```
//  BUTTON_Z => 0b00010000
// ~BUTTON_Z => 0b11101111
this.key_flag = 0b00010000 & 0b11101111;
```

AND演算子は、各bit それぞれについて、どちらか両方の値が 1 であれば、そのbitは1となります。
つまり、key_flag は `0b00000000` となります。

Zボタンと同時にXボタンも押されていればどうなるでしょうか

```
//  BUTTON_X => 0b00100000
//  BUTTON_Z => 0b00010000
// ~BUTTON_Z => 0b11101111
this.key_flag = 0b00110000 & 0b11101111;
```

既にZ, Xボタンが押されているので、`0b00110000` に対して AND 演算を行います。
結果は、`0b00100000` となり、Xボタンだけ押された状態は残ったまま、Zボタンが押された状態は解除されました。

```
// 指定のキーが押下状態か確認する
Input.prototype.isKeyDown = function(flag){
	return this.keyflag & flag;
};
```

`isKeyDown` メソッドは、指定されたキーが押下されているか確認するメソッドです。
`this.input.isKeyDown(Constant.BUTTON_Z)` のように呼び出します。

AND 演算子を使うことで、
該当のキーが押下されていれば `true` を、押下されてなければ、 `0x0` つまり `false` を返します。

`isKeyDown` は押下さえされていれば、キーを押しっぱなしでも常に `true` を返し続けます。
例えば自機のショットを撃つ際などの判定には向いていますが、
最初に押下された時に実行したいこと(メニューの決定など)の判定には向いていません。
(後者はで`isKeyDown` メソッドを使うと、毎フレーム決定が押下された扱いになってしまいます)
その場合は、`isKeyPush` メソッドを使用します。

```
// 指定のキーが押下されたか確認する
Input.prototype.isKeyPush = function(flag){
	// 1フレーム前に押下されておらず、現フレームで押下されてるなら true
	return !(this.before_keyflag & flag) && this.keyflag & flag;
};
```

指定のキーが最初に押下されたかどうかを判定します。
内部では、AND 演算子を用いて、該当のキーが現在のフレームで押下されており、
1つ前のフレームで押下されてなければ、true を返します。

この一つ前のフレームでの押下を判定するために、`before_keyflag` を使用しています。

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

最後に、キー入力を受け取るために、`Game` クラスを少し修正します。

## それ以外のキー情報の保存方法

キー情報の保存方法は、いくつか種類があります。前の項目で解説したのは、
「前の押下情報だけを保存しておく」方法でした。これは、キーが押下されているか、あるいはキーが押下されたかを
判定するだけのゲームであればこの方法で充分です。

他の保存方法には、「コマンド履歴を保存する」「押下時間も保存する」方法などがあります。
「コマンド履歴を保存する」方法は、格闘ゲームなどの、プレイヤーが入力したコマンド履歴をプログラムが参照したい場合に用いられます。
「押下時間も保存する」タイプは、例えばシューティングゲームのチャージショットなどを利用する場合に用いられます。

今回は解説しませんが、覚えておくと他のゲームの実装時にも応用が利くでしょう。

## ゲームパッド入力

前の項目で、プレイヤーのキーボード入力を取得する方法を紹介しました。
本項では、ゲームパッド(ゲームのコントローラー)からの入力も取得できるようにします。
HTML5 には GamePad API があるので、それを使ってゲームパッドからの入力を取得します。

`src/input.js`
```
Input.prototype.handleGamePad = function() {
	var pads = navigator.getGamepads();
	var pad = pads[0]; // 1Pコン

	if(!pad) return;

	this.keyflag = 0x00;
	this.keyflag |= pad.buttons[0].pressed ? constant.BUTTON_Z:      0x00;
	this.keyflag |= pad.buttons[1].pressed ? constant.BUTTON_X:      0x00;

	this.keyflag |= pad.axes[1] < -0.5 ? constant.BUTTON_UP:         0x00;
	this.keyflag |= pad.axes[1] >  0.5 ? constant.BUTTON_DOWN:       0x00;
	this.keyflag |= pad.axes[0] < -0.5 ? constant.BUTTON_LEFT:       0x00;
	this.keyflag |= pad.axes[0] >  0.5 ? constant.BUTTON_RIGHT:      0x00;
};
```

`Input` クラスの更新箇所です。`handleGamePad` 関数は、ゲームパッドからの入力を受け取って、
`keyflag` プロパティを更新するメソッドです。

`navigator.getGamepads` 関数により、ゲームパッドの情報を取得できます。`navigator.getGamepads` は、
配列を返します。複数のゲームパッドが接続されていた場合にもそれぞれのゲームパッドの値が取得できるように、
`navigator.getGamepads` は配列を返すようになっているのです。今回は1Pコントローラしか使用しないので、
`pads[0]` のみを取得します。

`pad` 変数には、`pad.buttons` と、`pad.axes` があります。buttons は各種ボタン、axes は各種方向キーです。
今回は、`buttons[0]` を、キーボードのZボタンと同等に、`buttons[1]` をキーボードのXボタンと同等に扱います。

`pad.axes` は、2つの値の配列であり、それぞれ -1 ~ 1 の値を取ります。
これは、スティックをどの方向にどれだけ倒したかによって強弱の概念があるからです。
今回は、0.5 と -0.5 を閾値として入力されたかされていないかをチェックしています。

`src/game.js`
```
Game.prototype.run = function() {
	this.input.handleGamePad();
	/* ~ 以下省略 ~ */
};
```

最後に、`Game` クラスの `run` メソッド内で、`handleGamePad` を呼び出します。
これで、ゲームプログラムから、ゲームパッドの値を取得することができます。
