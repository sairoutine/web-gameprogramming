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
```
// context
var audio_context = new AudioContext();

// mp3 ファイルの読み込み
var request = new XMLHttpRequest();
request.open('GET', "./path/to/file.mp3", true);
request.responseType = 'arraybuffer';

request.send();
request.onload = function () {
	var res = request.response;
	// arraybuffer を audio 再生用に decode
	audio_context.decodeAudioData(res, function (buffer) {
		// 音量調整用
		var audio_gain = audio_context.createGain();

		// 音量
		audio_gain.gain.value = 1.0;

		// source
		var source = audio_context.createBufferSource();

		// source に audio データを追加
		source.buffer = buffer;

		// ループ再生する
		source.loop = true;

		// ループの再生開始位置を指定
		source.loopStart = 5.23;

		// ループの再生終了位置を指定
		source.loopEnd = 10.55;

		// source → audio_gain を接続
		source.connect(audio_gain);

		// audio_gain → destination に接続
		audio_gain.connect(audio_context.destination);

		// 再生開始
		source.start(0);
		// 再生停止
		source.stop(0);
	});
};
```

sourceは、 再生するたびに生成して、使い捨てるというかたち になっている。

## 流れ
こちらの方法を利用する場合の, アルゴリズムの概要を記載します.

1. ArrayBuffer (バイナリデータ) を取得 (File API or Ajax)
2. ArrayBuffer (バイナリデータ) を参照 (decodeAudioDataメソッド)
3. 入力点を生成 (createBufferSourceメソッド)
4. 入力点と出力点を接続 (connectメソッド)
5. 音源のスイッチをON (startメソッド)


## コンテクスト
AudioContext(リアルタイムレンダリングの場合)とOfflineAudioContext(オフラインレンダリングの場合)が存在します。
これらは、BaseAudioContext インターフェースを継承しています。

OfflineAudioContext はレンダリング/ミックスダウンで使用される特殊な AudioContext であり、(可能性としては)リアルタイムよりも高速に動作します。 それはオーディオハードウェアに出力しない代わりに可能な限り高速に動作して、 Promise にレンダリング結果を AudioBuffer として戻します。


AudioContext インスタンスからは以下の Node を作ることができます。
ゲームで主に使うのは、gainNode と BufferNode くらいです。
```
AudioBufferSourceNode  createBufferSource ();
    Promise<AudioWorker>   createAudioWorker (DOMString scriptURL);
    ScriptProcessorNode    createScriptProcessor (optional unsigned long bufferSize = 0
              , optional unsigned long numberOfInputChannels = 2
              , optional unsigned long numberOfOutputChannels = 2
              );
    AnalyserNode           createAnalyser ();
    GainNode               createGain ();
    DelayNode              createDelay (optional double maxDelayTime = 1.0
              );
    BiquadFilterNode       createBiquadFilter ();
    IIRFilterNode          createIIRFilter (sequence<double> feedforward, sequence<double> feedback);
    WaveShaperNode         createWaveShaper ();
    PannerNode             createPanner ();
    SpatialPannerNode      createSpatialPanner ();
    StereoPannerNode       createStereoPanner ();
    ConvolverNode          createConvolver ();
    ChannelSplitterNode    createChannelSplitter (optional unsigned long numberOfOutputs = 6
              );
    ChannelMergerNode      createChannelMerger (optional unsigned long numberOfInputs = 6
              );
    DynamicsCompressorNode createDynamicsCompressor ();
    OscillatorNode         createOscillator ();
```

destination(AudioDestinationNode インスタンス) を持ち、単一の入力を持ち、全てのオーディオの最終的な出口を表しています。 

resume, suspend, close といったメソッドを持ち、これにより音の出力を止めたりできますが、
最終的な音の出口を止めるよりは、後述の source レベルで音の出力を止める方が有益です。


decodeAudioData
ArrayBuffer 内にあるオーディオファイルのデータを非同期にデコードします。

コールバック関数の引数は1つでデコードされた PCM オーディオデータをあらわす AudioBuffer になります。

## モジュラールーティング
![](./image/source_gain_destination.png)

異なる AudioNode オブジェクト同士を接続できます。それぞれのノードは入力／出力を持っています。

```
var source = audio_context.createBufferSource();
```

buffer_source は入力を持たず出力を持ちます。

```
audio_context.destination
```

destination は出力を持たず入力を持ちます。

```
// 音量調整用
var audio_gain = audio_context.createGain();
```

audio_gain は入力と出力を持ちます。
主に、入力されたデータの音量調整を行う加工を行い、それを出力します。

フィルターのような殆どの処理ノードは1つの入力と1つの出力を持ちます。

一つの Node に対して複数の Node を出力／入力することもできます。接続された Node 数はチャンネル数と呼ばれ、
チャンネル数は Node によって最大数がある場合もあります。

もしdestinationが他のAudioContextによって作成されたAudioNodeの場合、InvalidAccessError 例外を発生します(must)。 

disconnect(node) で切断することもできます。

buffer_sourceは
onended にイベントハンドラを設定することで、再生後をフックすることができます。
stopメソッドを実行, または, 再生時間が経過したときに発生するイベントハンドラ

start(when, offset) whenは context 上での再生開始時刻, offset はバッファ内の再生開始時刻です。

 AudioBufferSourceNodeインスタンスのstopメソッドを1度実行したあと, 再度startメソッドを実行しても再生されません. その理由は, AudioBufferSourceNodeは楽器音などのワンショットサンプルの再生に利用することを想定されているので, 使い捨てのノードという仕様になっているからです. したがって, オーディオを再開するには, 新たにAudioBufferSourceNodeインスタンスを生成する必要があります.

gainNode は一つを使いまわしても構いませんが、音量調整やフェードイン／フェードアウト設定をいちいちリセットしなくてはならないため、再生のたびに新しくつくることをオススメします。

## ガーベジコレクション
ガベージコレクション

AudioBufferSourceNodeは, 使い捨てのノードという仕様であることを解説しました. つまり, インスタンスの生成と破棄を繰り返すことが想定されています. したがって, AudioBufferSourceNodeインスタンスに割り当てられたメモリの解放が実行される条件については理解しておく必要があります.

AudioBufferSourceNodeインスタンスに限らず, Web Audio APIが定義するクラスのインスタンスにおいては, 以下の5つの条件すべてにあてはまる場合, ガベージコレクションの対象になります.

参照が残っていない
オーディオが停止している
サウンドスケジューリングが設定されていない
ノードが接続されていない
処理すべきデータが残っていない
つまり, 何らかの形で利用されているノードはガベージコレクションの対象とならないということです.

AudioBufferSourceNodeにおいて, ガベージコレクションが実行される条件で重要なのは, 最初の3つです. その理由は, AudioBufferSourceNodeは他のノードの出力先 (接続先) として利用されることがないこと, また, 最後の条件はConvolverNodeなどにおいて考慮すべきことであり, AudioBufferSourceNodeでは無関係だからです.

したがって, このセクションでは, 最初の3つの条件に関して解説します.

source.start(context.currentTime + counter++);


# タッチ時に再生を発火させとく
```
document.addEventListener('touchstart', this._onTouchStart.bind(this));
WebAudio._onTouchStart = function() {
    var context = WebAudio._context;
    if (context && !this._unlocked) {
        // Unlock Web Audio on iOS
        var node = context.createBufferSource();
        node.start(0);
        this._unlocked = true;
    }
};

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


gainNode.gain.linearRampToValueAtTime

WebAudio._fadeIn = function(duration) {
    if (this._masterGainNode) {
        var gain = this._masterGainNode.gain;
        var currentTime = WebAudio._context.currentTime;
        gain.setValueAtTime(gain.value, currentTime);
        gain.linearRampToValueAtTime(1, currentTime + duration);
    }
};

WebAudio._fadeOut = function(duration) {
    if (this._masterGainNode) {
        var gain = this._masterGainNode.gain;
        var currentTime = WebAudio._context.currentTime;
        gain.setValueAtTime(gain.value, currentTime);
        gain.linearRampToValueAtTime(0, currentTime + duration);
    }
};

