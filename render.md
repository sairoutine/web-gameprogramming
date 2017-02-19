# 描画
ここまでで、シーン及びシーン内にオブジェクトを追加しました。シーンやオブジェクトを追加したにも関わらず、画面には何も表示されずに寂しかったことと思います。いよいよ画面の描画について書いていきます。
## 画像の読み込み
まずは、画像をゲームに読み込む処理をしたいと思います。JavaScript は非同期なので、
画像を読み込みしている最中も処理が進行します。画像の読み込みをしている間は、
ゲームを進行させない(=ローディング画面を表示する)ようにしないと、画像が表示されないまま
ゲームが進行してしまいます。

`src/asset_loader/image.js`
```
var ImageLoader = function(game) {
	this.images = {};

	this.loading_image_num = 0;
	this.loaded_image_num = 0;
};
ImageLoader.prototype.loadImage = function(name, path) {
	var self = this;

	self.loading_image_num++;

	// it's done to load image
	var onload_function = function() {
		self.loaded_image_num++;
	};

	var image = new Image();
	image.src = path;
	image.onload = onload_function;
	this.images[name] = image;
};

ImageLoader.prototype.isAllLoaded = function() {
	return this.loaded_image_num > 0 && this.loaded_image_num === this.loading_image_num;
};

ImageLoader.prototype.get = function(name) {
	return this.images[name];
};
ImageLoader.prototype.remove = function(name) {
	delete this.images[name];
};


module.exports = ImageLoader;
```

```
var ImageLoader = function(game) {
	this.images = {};

	this.loading_image_num = 0;
	this.loaded_image_num = 0;
};
```

ImageLoader クラスのコンストラクタです。読み込んだ画像データを保持する `images` プロパティ、
読み込み中の画像の数である `loading_image_num`, 読み込んだ画像数である `loaded_image_num` を持ちます。

```
ImageLoader.prototype.loadImage = function(name, path) {
	var self = this;

	self.loading_image_num++;

	// it's done to load image
	var onload_function = function() {
		self.loaded_image_num++;
	};

	var image = new Image();
	image.src = path;
	image.onload = onload_function;
	this.images[name] = image;
};
```

画像の読み込みを開始します。

```
ImageLoader.prototype.isAllLoaded = function() {
	return this.loaded_image_num > 0 && this.loaded_image_num === this.loading_image_num;
};
```

画像の読み込みが完了したことをチェックするメソッドです。プログラム側で、画像データが必要になった場合、
まず `loadImage` メソッドにより、画像の読み込みを開始し、シーン側で、`isAllLoaded` 関数が true になるまで、
他のゲームの進行をストップする(=ローディング画面を表示する)必要があります。

```
ImageLoader.prototype.get = function(name) {
	return this.images[name];
};
```

読み込みした画像データを取得するメソッドです。

```
ImageLoader.prototype.remove = function(name) {
	delete this.images[name];
};
```

読み込んだ画像を破棄するメソッドです。小さなゲームであれば、画像を破棄する必要はありませんが、
大きな規模のゲームでは、必要のなくなった画像は破棄しなくては、メモリを大量に消費してしまいます。

## スプライト

スプライトとは、複数の画像データを一つの画像ファイルにまとめたものです。同じ画像データでも、複数の画像ファイルを読み込むのと、1つのファイルで読み込んで、プログラム内で分割して使うのとでは、後者の方が読み込み速度が早くなります。

一般的に、キャラのアニメーションなどに使用する画像ファイルや、敵の弾の種類は、一つの画像ファイルにまとめる傾向があります。

プログラムでは、スプライト画像を読み込み、指定の領域のみを切り取って表示することで使用します。

それでは、Chara クラスにスプライトを使用して、画像を表示するようにしてみましょう。
いくつかのスプライト画像の表示に使う定数の追加と、draw メソッドの変更を行います。

# TODO:
image_loader.loadImage を読み出してない

**スプライト画像**  
![早苗](./image/player_sanae.png)

http://danmakufu.wiki.fc2.com/wiki/%E7%B4%A0%E6%9D%90%E3%83%AA%E3%83%B3%E3%82%AF
星蓮船自機ドット絵
霊夢・魔理沙・早苗の星蓮船風ドット絵
よりお借りしております。

`src/object/chara.js`
```
// 描画
Chara.prototype.draw = function () {
	var image = this.core.image_loader.getImage(this.spriteName());

	var ctx = this.core.ctx;

	ctx.save();

	// set position
	ctx.translate(this.x, this.y);

	var sprite_width  = this.spriteWidth();
	var sprite_height = this.spriteHeight();

	ctx.drawImage(image,
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

Chara.prototype.spriteName = function(){
	return "sanae";
};
Chara.prototype.spriteIndexX = function(){
	return 0;
};
Chara.prototype.spriteIndexY = function(){
	return 0;
};
Chara.prototype.spriteWidth = function(){
	return 64;
};
Chara.prototype.spriteHeight = function(){
	return 64;
};
module.exports = Chara;
```

先ほど紹介した Chara クラスを改良しました。それではコードを追っていきましょう。

```
// 描画
Chara.prototype.draw = function () {
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
``

HTML5 の Canvas API を利用して、スプライトの描画を行っています。
オブジェクトから Canvas の API を利用するには、`this.core.ctx` を参照します。

```
var image = this.core.image_loader.getImage(this.spriteName());
```

読み込んだ画像を取得しています。

```
	ctx.translate(this.x, this.y);
```

Canvas 上のどこに画像を描画するかを指定しています。

```
	ctx.drawImage(
		image,
		// sprite position
		0,                                  0,
		// sprite size to get
		sprite_width,                       sprite_height,
		// adjust left x, up y because of x and y indicate sprite center.
		-sprite_width/2,                    -sprite_height/2,
		// sprite size to show
		sprite_width,                       sprite_height
	);
```

取得した画像を drawImage 関数を利用して Canvas に描画しています。
drawImage 関数は、引数に指定した数によって挙動が変わる関数です。今回は引数が9つの使い方をします。

第1引数の image には、描画したい画像データを指定します。

第2引数・第3引数には、画像のどこからを描画したいかを x, y 座標で指定します。
画像の左上が(0, 0)となるので、今回は (0, 0)を指定することにします。

第4引数,第5引数には、Canvas 上のどこに画像を描画するか、x, y 座標で指定します。
`translate` で指定した描画位置から、相対的に描画位置をずらします。
これは、`this.x` と `this.y` が指定する画像の位置は、基準を画像の真ん中にするためです。

`translate` で描画位置を指定し、さらに drawImage で描画位置をずらすのは二度手間に見えるかもしれませんが、
のちのち、画像の回転を行う上で大切になってきます。また画像の回転の項目で後述します。

```
Chara.prototype.spriteName = function(){
	return "sanae";
};
Chara.prototype.spriteWidth = function(){
	return 64;
};
Chara.prototype.spriteHeight = function(){
	return 64;
};
```

`spriteName` は読み込んだ画像を表示する定数です。`spriteWidth` `spriteHeight` には、
読み込んだ画像のどこまでを切り取るか、横幅と高さを指定します。

# TODO: 
ここにスプライトの基準点と、切り取りについて解説した画像を出す





## スプライトアニメーション

先ほど、スプライト画像からスプライトを切り取って表示することを行いました。
これにより自機のキャラ画像が表示されたことかと思います。

さらに自機の画像をアニメーションさせてみたいと思います。

アニメーションの方法は、パラパラ漫画と同じように、
N秒毎に、表示するスプライトを変更することで行います。

それでは、自機のスプライト画像2種類を、を10フレーム毎に切り替えて、
アニメーションさせてみましょう。 Chara クラスを変更します。


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

	// 現在表示するスプライト
	this.current_sprite_index = 0;
};
Chara.prototype.id = function () {
	return this._id;
};
// 更新
Chara.prototype.update = function () {
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
Chara.prototype.draw = function () {
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

Chara.prototype.spriteName = function(){
	return "sanae";
};
Chara.prototype.spriteIndexX = function(){
	return this.spriteIndices()[this.current_sprite_index].x;
};
Chara.prototype.spriteIndexY = function(){
	return this.spriteIndices()[this.current_sprite_index].y;
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


```
var Chara = function (scene) {
	this.scene = scene;
	this.game = game;

	this._id = "chara";

	this.x = 0;
	this.y = 0;

	// 経過フレーム数
	this.frame_count = 0;

	// 現在表示するスプライト
	this.current_sprite_index = 0;
};
```

`this.current_sprite_index` というプロパティを追加しました。これは、
現在表示するスプライトがどれかを表します。今回は、スプライト画像が2種類なので、0 か 1 の値を取り、
また、10フレームごとに変更します。

```
Chara.prototype.spriteAnimationSpan = function(){
	return 10;
};
```

スプライトを切り替える間隔を定義しています。10 フレームごとに切り替えるので、10を指定しました。

```
Chara.prototype.spriteIndices = function(){
	return [{x: 0, y: 0},{x: 1, y: 0}];
};
```

`spriteIndices` にスプライト画像の位置を指定しています。今回は2種類のスプライト画像を使用するので、
左から0番目、上から0番目のスプライト画像を1つと、左から1番目、上から0番目のスプライト画像を表示するので、
`[{x: 0, y: 0},{x: 1, y: 0}]` を指定しました。

```
Chara.prototype.spriteIndexX = function(){
	return this.spriteIndices()[this.current_sprite_index].x;
};
Chara.prototype.spriteIndexY = function(){
	return this.spriteIndices()[this.current_sprite_index].y;
};
```

current_sprite_index と spriteIndices を元に、スプライト画像のどの座標を切り取ればいいのかを返します。

```
// 描画
Chara.prototype.draw = function () {
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
```

drawImage の第2引数と第3引数が変更になっております。`spriteIndexX` と `spriteIndexY` を元に、
スプライト画像のどこを切り取るかを指定しています。この `spriteIndexX` と `spriteIndexY` は
先述した `this.current_sprite_index` を元に決定されます。

```
// 更新
Chara.prototype.update = function () {
	this.frame_count++;

	// animation sprite
	if(this.frame_count % this.spriteAnimationSpan() === 0) {
		this.current_sprite_index++;
		if(this.current_sprite_index >= this.spriteIndices().length) {
			this.current_sprite_index = 0;
		}
	}

};
```

先述した、表示するスプライトを切り替える処理です。spriteAnimationSpan 毎に、current_sprite_index を +1 しており、
また、表示できるスプライトがなくなれば、また最初のスプライトに戻ります。

これで、自機キャラがアニメーションするようになりました。
