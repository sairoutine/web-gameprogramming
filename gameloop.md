# ゲームループ
Webエンジニアが初めてゲームを作るときにまず戸惑うのが「ゲームループ」という考え方だと思います。
本稿では、ゲーム画面上で、キャラクターの動きがどのようにして更新されていくのかを、「ゲームループ」という考え方でご説明させていただきます。

FPS(frame per second)という言葉を聞いたことがありますでしょうか。「60FPS」「30FPS」という言葉を聞いたことがあるかと思います。60FPSという言葉について解説します。「60FPS」のゲームでは、1秒間に60回、画面が更新されています。つまり、1/60秒ごとに少しずつ違う画面を計算・描画することで、私たちの目にゲームが「動画」として映っているのです。下記のコードを見てみてください。

```
function update() {
	// ゲーム更新
}

function run() {
	update();

	requestAnimationFrame(run);
}

run();
```

`requestAnimationFrame` メソッドは、ブラウザの次の再描画の直前に、指定した関数を呼び出すようにブラウザに要求します。
ブラウザの再描画は基本的に1秒に60回行われるため、上記のコードを書くと、1秒間に60回、update 関数が呼び出されます。

この update 関数の中で、ゲームの描画であったり、描画に必要な情報の更新を行います。

# 処理落ちとフレーム落ち
ゲームループは2種類の実装方法があります。これは、処理に時間がかかった際に「処理落ち」させるか「フレーム落ち」させるかの違いです。

1秒間に60回処理する場合、最大16.666ms以内に1処理を終わらせないといけません。
それ以上時間、処理に時間をかけた場合に、「処理を遅延させるか」「描画をスキップするか」が
処理落ちとフレーム落ちの違いです。

「処理落ち」が発生すると、画面の動きがすごくゆっくりになります。画面の動きはゆっくりになりますが、描画はスキップしません。
弾幕シューティングゲームなどは描画がスキップされると、プレイヤーが理解できないまま自機が死んでしまいます。描画がスキップされると困る場合には、処理落ちする方のゲームループを使用します。先ほどのサンプルコードのゲームループは、処理落ちするゲームループです。

一方「フレーム落ち」が発生すると、画面の動きがいきなり飛んだように見えます。処理自体は時間経過どおりに進みます。
そのフレームの描画を諦めて次のフレームの描画に移るため、画面の動きがカクカクっと飛んだ感じになります。
ただ実際の時刻との同期は守られるため、動き自体が遅くなることはありません。

処理落ちとフレーム落ちでは、フレーム落ちの方が実装がちょっと面倒くさいです。
フレーム落ちさせる場合のコードは以下となります。
```
var deltaTime = 1.0 / 60.0;
var currentTime;
var accumulator;

function update() {
	// ゲーム更新
}

function run() {
	var newTime = performance.now();
	var fTime = (newTime - currentTime) / 1000;

	// 追いつくための実行回数があまりにも増えないようにする
	if (fTime > 0.25) fTime = 0.25;
	var currentTime = newTime;
	accumulator += fTime;

	// 実行回数を増やして処理を追いつかせる
	while (accumulator >= deltaTime) {
		update();
		accumulator -= deltaTime;
	}
	requestAnimationFrame(run);
}

run();

```

描画毎に、前回の実行時との経過時間を計算し、1/60 ms 以内に処理が収まってなければ、
次の実行時に複数回実行することで、処理を追いつかせています。

# Canvas
それでは、Canvas API を使って、試しに描画をしてみましょう。
`public/index.html`
```
<html>
	<canvas id="mainCanvas" width="640" height="480"></canvas>
</html>
```

index.html に `canvas` タグを追加します。ここにゲーム内容を描画します。`canvas` タグの id 属性は `mainCanvas` です。あとで JavaScript 側から Canvas の DOM を取得する際に使用します。

先ほどのコードでcanvas タグに文字を表示してみたいと思います。

```
var mainCanvas = document.getElementById('mainCanvas');
var context = mainCanvas.getContext('2d');
// ゲーム更新
function update() {
	// 画面をクリア
	context.clearRect(0, 0, mainCanvas.width, mainCanvas.height);
	context.font = "18px 'MS ゴシック'";
	context.fillStyle = 'rgb( 255, 255, 255 )'; // 黒

	// 文字を描画
	context.fillText("Hello World!", mainCanvas.width/2, mainCanvas.height);

}

function run() {
	update();

	requestAnimationFrame(run);
}

run();
```

画面中央に「Hello World!」という文字列が表示されたかと思います。

# 終わりに

それでは実際のゲームを作っていきたいと思います。まずゲームクラスを作っておきます。
ゲームクラスに、ゲームループに必要な処理を実装することにします。

`src/game.js
```
var Game = function (canvas) {
	this.ctx = canvas.getContext('2d'); // Canvas への描画は、ctx プロパティを通して行うこととする。
	// 画面サイズ
	this.height = canvas.height;
	this.width = canvas.width;

};
Game.prototype.startRun = function () {

	this.run();
};

Game.prototype.run = function () {
	// ここにゲームの処理を書く

	this.request_id = requestAnimationFrame(this.run.bind(this));
};
module.exports = Game;
```

`src/main.js` はゲームのエントリポイント(最初の開始地点)です。
ゲームクラスのインスタンスを生成し、ゲームを起動します。

`src/main.js`
```
var Game = require("game");
var mainCanvas = document.getElementById('mainCanvas');
var game = new Game(mainCanvas);
game.startRun();
```
