# BGM/SE再生

Web上での音声の再生は、HTML5 Audio を使う方法と、Web Audio を使う方法の二種類が存在します。

HTML5 Audio は、いわゆる<audio>タグを作成し、それを利用して、再生します。
簡単に音声が再生できる代わりに、細かい制御ができません。

Web Audio は、JavaScript の API です。細かい制御ができる代わりに、
知らなければならないことが増えます。

# HTML5 Audio
```
var element = new Audio();
element.addEventListener('canplay', function(e) {
	// 音声データの再生が可能になったら
});
element.src = "./path/to/file.mp3"; // 音声ファイルの指定
element.volume = 1.0; // 音量 0.0 ~ 1.0
element.loop = false; // ループするか否か
element.currentTime = 0; // 再生位置の指定
element.load(); // src で指定した音声ファイルの読み込みを開始する
element.pause(); // 再生の停止
element.play(); // 再生の停止
```

HTML5 Audio の使い方は上記のようになります。
ブラウザの種類にもよりますが、mp3 や wav, ogg, m4a 等が再生できます。

HTML5 Audio は非常に簡単に音声ファイルの再生を行うことができますが、
BGMのループの開始位置／終了位置を指定することができません。

そのため、ちょっとややこしい概念は出てきますが、Web Audio を使用して、
BGMやSEを再生したいと思います。

# Web Audio




## 音声の読み込み
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
## SEを再生するときは一旦フラグ立てる
## フェードイン／フェードアウト


