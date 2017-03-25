# その他

## FPS 計算

開発時には、FPSを表示しておくと便利です。
弾幕シューティングゲームでは、処理落ちが発生することがよくあるので、
どこで処理落ちしているのか、どのくらい重いのかを判断することができます。

```
var FPS_SPAN = 30;
var DEBUG = 1;

var Game = function () {
	/* ~ 省略 ~ */

	// 前回にFPSを計算した際の時刻(ミリ秒)
	this._before_calc_fps_time = 0;

	// 計測したFPS
	this._fps = 0;

	/* ~ 省略 ~ */
};

Game.prototype.run = function () {
	if(DEBUG) { // デバッグの際のみFPSを表示する。
		this._calcFPS();
		this._renderFPS();
	}
};
Game.prototype._calcFPS = function () {
	// FPS_SPAN 毎にFPSを計測する
	if((this.frame_count % FPS_SPAN) !== 0) return;

	// 現在時刻(ミリ秒)を取得
	var newTime = Date.now();

	if(this._before_calc_fps_time) {
		this._fps = parseInt(1000 * FPS_SPAN / (newTime - this._before_calc_fps_time));
	}

	this._before_calc_fps_time = newTime;
};
Game.prototype._renderFPS = function () {
	// FPSをレンダリング
	var ctx = this.ctx;
	ctx.save();
	ctx.fillStyle = 'rgb( 6, 40, 255 )';
	ctx.textAlign = 'left';
	ctx.fillText("FPS: " + this._fps, this.width - 70, this.height - 10);
	ctx.restore();
};
```

1フレームごとに、処理にどれくらい時間がかかったかを計算すると、
FPSの計算自体に処理時間を使ってしまうので、30フレームごとに計算することにします。
30フレーム経過する時間をマイクロ秒単位で計算し、1秒で何フレーム処理できたかを表示しています。

## セーブ
ゲームのデータをクライアント側で保存する際は、localStorage を利用すると便利です。
localStorage とは WebStorage の一種で、JavaScript を用いてクライアント側にデータを保存する仕組みです。
ユーザーのデータを半永久的にデータを保持することができます。
データの読み込み・更新も比較的簡単です。

データの保存
```
var SaveStorage = function () {
	throw new Error("this class is static class");
}

SaveStorage.key = "save_key";

SaveStorage.save = function(data) {
	var self = this;
	localStorage.setItem(this.key, JSON.stringify(data));
};
```

データの読み込み
```
SaveStorage.load = function() {
	return JSON.parse(localStorage.getItem(this.key));
};
```

localStorage には文字列しか保存できないため、JavaScript のオブジェクトを保存できるように、
`JSON.stringify`, `JSON.parse` を使用しています。

# 画面最大化

```
window.changeFullScreen = function () {
	var mainCanvas = document.getElementById('mainCanvas');
	if (mainCanvas.requestFullscreen) {
		mainCanvas.requestFullscreen();
	}
};
```

`public/index.html`
```
<html>
<head>
<style>
#mainCanvas:-webkit-full-screen {
	height: 100%;
	width:  100%;
}
</style>
</head>
<body>
	<canvas id="mainCanvas" width="640" height="480"></canvas>
	<input type="button" value="最大化" onclick="changeFullScreen();"><br />
	<script type="text/javascript" src="./js/main.min.js"></script>
</body>
</html>
```

Canvas の DOM の requrestFullscreen メソッドを呼び出すことで、
ゲーム画面を最大化することができます。

また、Canvas DOM の `-webkit-full-screen` に CSSを当てています。
これは、フルスクリーン時に適用されるCSSです。
今回は height と width を 100% にすることで、画面最大化時に、ゲーム画面も最大化して表示しています。
(これがないと、画面は最大化しても、ゲーム画面は 640 x 480px のままになります)

# Electron

