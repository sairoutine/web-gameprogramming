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

Electron は JavaScript でデスクトップアプリケーションが作成できるツールです。
本書では、JavaScript と HTML5 を利用して、Webブラウザ上で動くゲームを作成しましたが、
Electron を使用することで、Web ブラウザ上で動くゲームをPC向けアプリケーションとして出力することができます。

npm で electron と electron-packager をインストールします。
`electron` は、PCでの実行環境のことで、`electron-packager` は、
実際に Web アプリケーションを、PC向けアプリとしてビルドする際に使用するツールです。

```
npm install --save-dev electron electron-packager
```

Electron 上で Webブラウザゲームを動かすスクリプトを作ります。

`public/electron.js`
```
'use strict';
var electron = require('electron');
var app = electron.app;
var dialog = electron.dialog;
var BrowserWindow = electron.BrowserWindow;

var mainWindow;

function createWindow () {
	mainWindow = new BrowserWindow({
		"width":          640,
		"height":         480,
		"useContentSize": true,  // フレームのサイズをサイズに含まない
		"resizable":      false, // ウィンドウのリサイズを禁止
		"alwaysOnTop":    true,  // 常に最前面
	});

	mainWindow.loadURL(`file://${__dirname}/index.html`);

	mainWindow.on('closed', function () {
		mainWindow = null;
	});
}

app.on('ready', createWindow);

app.on('window-all-closed', function () {
	if (process.platform !== 'darwin') {
		app.quit();
	}
});

app.on('activate', function () {
	if (mainWindow === null) {
		createWindow();
	}
});
```

640 × 480 のウィンドウを作成し、そこに index.html を表示しています。
Electron にはメインプロセスとレンダラプロセスの概念がありますが、
このように Webブラウザ上で既に動くゲームをPC向けアプリケーションとして起動する場合は、
レンダラプロセス上で index.html を実行するだけで、PC向けゲームを作成できます。


package.json を少し更新します。以下の行を `package.json` に追加します。

`./package.json`
```
"main": "electron.js",
"scripts": {
	"start": "electron ./public/electron.js",
	"build:win": "electron-packager ./public AppName --platform=win32  --arch=x64 --icon=icon.ico",
	"build:mac": "electron-packager ./public AppName --platform=darwin --arch=x64 --icon=icon.icns",
},
```

`main` に指定したファイルが、Electron で作成したPC向けアプリケーションが
起動する際に実行される JavaScript ファイルです。

ビルド前に、開発用として Electron で実行できるように、`npm start`コマンドを定義しています。

また 64bit の Windows と Mac 向けに `npm run-script build:win` と `npm run-script build:mac` コマンドを定義しています。
これらは、electron-packager によりビルドされます。

```
electron-packager <sourcedir> <appname> --platform=<platform> --arch=<arch> [optional flags...]
```

`electron-packager` コマンドの使い方は上記のようになります。`sourcedir` がビルド対象の html や js があるディレクトリです。
minify 前の js は必要ないので、 今回は public ディレクトリを指定しました。

`appname` はアプリケーションの名前です。 `platform` は Windows や Mac あるいは Linux などどのOS向けにビルドするのかを指定します。`arch` は 32bit もしくは 64bit 向けに出力するのかを指定します。`icon` には、アプリケーションのアイコンを指定します。Windows と Mac では、使用できるアイコンのフォーマットが異なるので、2種類用意することに注意してください。

## TIPS
Mac 向けのアプリケーションをビルドする際、リサイズしない設定にしていても、
ズームを行うことが可能になったりします。ズームを禁止させるには、クライアント側のコード(`main.js` など)に下記のコードを追加します。

```
if (typeof window !== 'undefined' && window.process && window.process.type === 'renderer') {
	require('electron').webFrame.setZoomLevelLimits(1,1); //zoomさせない
}
```


