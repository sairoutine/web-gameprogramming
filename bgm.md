# BGM/SE再生

これまでの項で、シューティングゲームのゲームシステムに必要な最低限の実装は一通り紹介しました。
本項では、さらにBGM や SE の再生について解説します。

Web上での音声の再生は、HTML5 Audio を使う方法と、Web Audio を使う方法の二種類が存在します。

HTML5 Audio は、いわゆる`<audio>`タグを生成し、それを利用して、再生します。
簡単に音声が再生できる代わりに、再生の細かい制御ができません。

Web Audio は、JavaScript から利用する API です。細かい制御ができる代わりに、
制御に必要な細かい概念を知らなければなりません。

## HTML5Audio

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

HTML5 Audio の使い方は上記のようになります。`Audio` インスタンスを作成し、`play` メソッドで再生します。
再生できるフォーマットは、ブラウザの種類にもよりますが、mp3 や wav, ogg, m4a 等が再生できます。

HTML5 Audio は非常に簡単に音声ファイルの再生を行うことができますが、
BGMのループの開始位置／終了位置を指定することができません。

そのため、ちょっとややこしい概念は出てきますが、Web Audio を使用して、
BGMやSEを再生したいと思います。

## WebAudio

```
// context 作成
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
		// 音量調整用の GainNode 作成
		var audio_gain = audio_context.createGain();

		// 音量
		audio_gain.gain.value = 1.0;

		// オーディオデータの出力を行う AudioBufferSourceNode 作成
		var source = audio_context.createBufferSource();

		// AudioBufferSourceNode に オーディオデータを追加
		source.buffer = buffer;

		// ループ再生する
		source.loop = true;

		// ループの再生開始位置を指定
		source.loopStart = 5.23;

		// ループの再生終了位置を指定
		source.loopEnd = 10.55;

		// AudioBufferSourceNode → GainNode を接続
		source.connect(audio_gain);

		// GainNode → destination に接続
		audio_gain.connect(audio_context.destination);

		// 再生開始
		source.start(0);

		// 再生停止
		source.stop(0);
	});
};
```

Web Audio の使い方は上記のようになります。

### 流れ

ゲームにおいて Web Audio を使用する際の基本的な流れは以下になります。
オーディオデータの加工やミキシング等を行うには
より複雑な流れが必要ですが、BGMの再生やSEの再生程度であれば、
下記の流れで充分です。

0. コンテクスト(AudioContext) を生成
1. オーディオデータ(ArrayBuffer) を取得(Ajax)
2. オーディオデータからPCMオーディオデータを生成
3. オーディオデータの入力点(AudioBufferSourceNode)を生成
3. 音量調整ノード(GainNode)を生成
4. オーディオデータと音量調整ノードを接続
5. 音量調整ノードと出力を接続
6. 音源のスイッチをON

### コンテクスト
Web Audioで何かする前に、まずコンテクストを作成する必要があります。
コンテキストはオーディオ処理やデコードを行うノードの作成を制御します。
(ノードについては後ほど説明します。)

コンテクストには、`AudioContext`(リアルタイムレンダリング)と`OfflineAudioContext`(オフラインレンダリング)が存在しますが、
ゲームにおいては、`AudioContext` を使います。

`AudioContext` インスタンスからは以下のノードを作ることができます。

- AudioBufferSourceNode
 - GainNode
 - ScriptProcessorNode
 - AnalyserNode
 - DelayNode
 - BiquadFilterNode
 - IIRFilterNode
 - WaveShaperNode
 - PannerNode
 - SpatialPannerNode
 - StereoPannerNode
 - ConvolverNode
 - ChannelSplitterNode
 - ChannelMergerNode
 - DynamicsCompressorNode
 - OscillatorNode

たくさんの種類がありますが、ゲームで主に使うのは、`GainNode` と `AudioBufferSourceNode` くらいです。
この2つについては、ノードの項で説明します。

### ノード

ノードとは、音声データのフィルターのようなものです。入力と出力を持ちます。
また、複数のノードを、入力と出力で接続して使います。
ノードの種類には例えば、入力された音声データにディレイをかけて出力したり、
入力された音声データの音量調整を行って出力したり、というのがあります。

一つのノードに対して複数のノードの出力／入力を接続することもできます。
接続されたノードの数はチャンネル数と呼ばれ、チャンネル数はノードの種類によって最大数がある場合もあります。

ゲームで主に使うのは、`GainNode` と `AudioBufferSourceNode`、`AudioDestinationNode` です。
それぞれ、以下の役割があります。

 - AudioBufferSourceNode はオーディオデータのバイナリを出力します。
 - GainNode は音量調整を行うノードです。
 - AudioDestinationNode はオーディオの最終的な出口です。

### モジュラールーティング
![](./image/source_gain_destination.png)  
*Web Audio API の基礎(https://www.html5rocks.com/ja/tutorials/webaudio/intro/) より引用*

ノードの項目でも説明しましたが、ノードは入力と出力を持ち、
あるノードの出力と、別のノードの入力を接続することができます。

今回は、ゲームにおいて Web Audio を使用するので、以下のような
ノードの接続の仕方をすることにします。

AudioBufferSourceNode -> GainNode -> AudioDestinationNode

それでは 1つずつ解説します。

```
var source = audio_context.createBufferSource();
```

`source` は `AudioBufferSourceNode` インスタンスです。入力を持たず出力を持ちます。

```
source.buffer = buffer;
```

`AudioContext.decodeAudioData` メソッドにより PCMデータに変換したオーディオデータを代入します。

```
source.loop = true;
source.loopStart = 5.23;
source.loopEnd = 10.55;
source.connect(audio_gain);
```

`AudioBufferSourceNode` インスタンスには、ループの再生や、ループ再生の開始／終了位置を設定することができます。

`AudioBufferSourceNode` インスタンスは使い捨てです。一度あるBGMやSEの再生に使用した後、
別のBGMやSEの再生に使用することはできません。BGMやSEの再生のたびに `AudioBufferSourceNode` インスタンスを
生成することになります。

```
var audio_gain = audio_context.createGain();
audio_gain.gain.value = 1.0; // 音量を 1.0 に設定
```

`audio_gain` は `GainNode` インスタンスです。入力と出力を持ちます。
入力されたデータの音量調整を行う加工を行い、それを出力します。

`GainNode` は一つのインスタンスを使いまわして、複数のBGMやSEを再生しても構いませんが、
音量調整やフェードイン／フェードアウト設定をいちいちリセットしなくてはならないため、
再生のたびに新しくインスタンスを生成することをオススメします。

```
audio_context.destination
```

`destination` は `AudioDestinationNode` インスタンスです。出力を持たず入力を持ちます。
オーディオの最終的な出力先です。

```
// AudioBufferSourceNode → GainNode を接続
source.connect(audio_gain);

// GainNode → destination に接続
audio_gain.connect(audio_context.destination);
```

各ノードのインスタンスが持つ `connect` メソッドにより、各ノードの入力と出力を接続することができます。

```
// 再生開始
source.start(0);

// 再生停止
source.stop(0);
```

`AudioBufferSourceNode` インスタンスの `start`, `stop` メソッドにより、再生の開始／停止ができます。
(`AudioDestinationNode` ではなく、`AudioBufferSourceNode` です。)

引数の 0 は、今すぐ再生するという意味です。N秒後に再生したい場合、引数に開始時刻を渡します。

## iOS端末での再生

iOS ではユーザーのアクション契機でないとオーディオを再生できません。
一度ユーザーのアクション契機で再生すれば、以降はユーザーのアクション契機でなくとも
オーディオを再生することができます。

よって、iOS向けの端末でのゲームでは、ユーザーの最初のタッチ時に
無音のオーディオを再生する処理をすると便利です。

```
var unlocked = false;

document.addEventListener('touchstart', function() {
	if (unlocked) {
		var node = audio_context.createBufferSource();
		node.start(0);
		unlocked = true;
	}
};
```

## オーディオの読み込み

それでは、Web Audio でのオーディオデータの再生について解説しましたので、
いよいよそれを使って、ゲーム内で BGM や SEを再生する仕組みを整えていきたいと思います。

`src/asset_loader/audio.js`
```
var AudioLoader = function(game) {
	this.context = new AudioContext();
	this.audios = {}; // オーディオデータ

	this.loading_audio_num = 0;
	this.loaded_audio_num  = 0;

	this.playing_bgm_source = null; // 現在再生中のBGMの AudioBufferSourceNode インスタンス
	this.playing_bgm_gain   = null; // 現在再生中のBGMの GainNode インスタンス
	this.next_play_bgm      = null; // 次のフレームで再生予定のBGM
	this.next_play_se       = {};   // 次のフレームで再生予定のSE
};
AudioLoader.prototype.loadAudio = function(name, path, volume, is_loop, loop_start, loop_end) {
	var self = this;

	volume = volume || 1.0;

	self.loading_audio_num++;

	var request = new XMLHttpRequest();
	request.open('GET', path, true);
	request.responseType = 'arraybuffer';

	request.send();
	request.onload = function () {
		var res = request.response;
		// arraybuffer を audio 再生用に decode
		audio_context.decodeAudioData(res, function (buffer) {
			self.audios[name] = {
				buffer:     buffer,
				volume:     volume,
				is_loop:    is_loop,
				loop_start: loop_start,
				loop_end:   loop_end,
			};

			self.loaded_audio_num++;
		});
	};
};


AudioLoader.prototype.isAllLoaded = function() {
	return this.loaded_audio_num > 0 && this.loaded_audio_num === this.loading_audio_num;
};

// SE の再生
AudioLoader.prototype.playSE = function(name) {
	this.next_play_se[name] = true;
};
// BGM の再生
AudioLoader.prototype.playBGM = function(name) {
	this.next_play_bgm = name;

};

// SE, BGM の再生の実行
AudioLoader.prototype.executePlay = function() {

	// SE の再生
	for (var name in this.next_play_se) {
		var source = this._setupBufferSource(name);
		var audio_gain = this._setupAudioGain(name);

		source.connect(audio_gain);
		audio_gain.connect(this.context.destination);

		// 再生開始
		source.start(0);
	});

	// BGM の再生
	if (this.next_play_bgm) {
		this.playing_bgm_source = this._setupBufferSource(this.next_play_bgm);
		this.playing_bgm_gain   = this._setupAudioGain(this.next_play_bgm);

		this.playing_bgm_source.connect(this.playing_bgm_gain);
		this.playing_bgm_gain.connect(this.context.destination);

		// 再生開始
		this.playing_bgm_source.start(0);
	});

	// reset
	this.next_play_se = {};
	this.next_play_bgm = null;
};

// BGM の再生停止
AudioLoader.prototype.stopBGM = function() {
	if(this.playing_bgm_source) {
		this.playing_bgm_source.stop(0);

		// reset
		this.playing_bgm_source = null;
		this.playing_bgm_gain   = null;
	}
};

AudioLoader.prototype._setupBufferSource = function(name) {
	var data = this.audios[name];

	var source = this.context.createBufferSource();
	source.buffer = data.buffer;

	if(data.is_loop) {
		source.loop = true;
	}

	if(data.loop_start) {
		source.loopStart = loop_start;
	}

	if(data.loop_end) {
		source.loopEnd = loop_end;
	}

	return source;
};
AudioLoader.prototype._setupAudioGain = function(name) {
	var data = this.audios[name];

	var audio_gain = this.context.createGain();
	audio_gain.gain.value = data.volume;

	return audio_gain;
};

AudioLoader.prototype.remove = function(name) {
	delete this.audios[name];
};

module.exports = AudioLoader;
```

上記が、`AudioLoader` クラスの全容です。それではまた少しずつ解説していきたいと思います。

```
// SE の再生
AudioLoader.prototype.playSE = function(name) {
	this.next_play_se[name] = true;
};
// BGM の再生
AudioLoader.prototype.playBGM = function(name) {
	this.next_play_bgm = name;

};
```

ゲーム内で、`playSE` 及び `playBGM` メソッドを呼び出すことで、オーディオを再生します。
ただし、メソッドの中身を見てもらえばわかるように、`playSE` 及び `playBGM` 内では
再生予定のオーディオのフラグを立てているだけです。

これは、1フレーム内に同じオーディオが複数再生される場合に、
1度しか再生しないようにするためです。例えば、シューティングゲームであれば、
複数の敵が同時に死ぬ場合がありますが、この際に死んだ敵の回数だけ敵が死ぬ音が再生されると
音が大きくなるあるいは、再生タイミングによってはそれぞれの SE がそれぞれズレてしまうため、
このように再生するときは一旦フラグだけ立てて、後述する `executePlay` メソッドで実際の再生を行います。

```
// SE, BGM の再生の実行
AudioLoader.prototype.executePlay = function() {
	/* ~ 省略 ~ */

	// BGM の再生
	if (this.next_play_bgm) {
		this.playing_bgm_source = this._setupBufferSource(this.next_play_bgm);
		this.playing_bgm_gain   = this._setupAudioGain(this.next_play_bgm);

		this.playing_bgm_source.connect(this.playing_bgm_gain);
		this.playing_bgm_gain.connect(this.context.destination);

		// 再生開始
		this.playing_bgm_source.start(0);
	});

	/* ~ 省略 ~ */
};
```

`executePlay` 関数では、`playSE` `playBGM` メソッドによって、
フラグの立っている SE や BGM の再生を実際に行います。
BGMの再生のときのみ、`AudioBufferSourceNode` インスタンスを変数に保存しています。
これは、`stopBGM` メソッドによって、BGMの再生を後ほど停止できるようにするためです。

`executePlay` メソッドは `run` 関数内で一度だけ呼び出してください。

```
Game.prototype.run = function () {
	/* ~ 省略 ~ */
	this.audio_loader.executePlay();
};
```

## フェードイン／フェードアウト

ゲームでは、BGMのフェードインやフェードアウトを実装することがあると思います。
これは、`GainNode` インスタンスの `gain` プロパティに存在する `linearRampToValueAtTime` メソッドを使用すると、
BGM のフェードイン／フェードアウトを簡単に実装することができます。

`src/asset_loader/audio.js`
```
AudioLoader.prototype.fadeInBGM = function(duration) {
	var self = this;
	if(self.playing_bgm_gain) {
		var gain = self.playing_bgm_gain.gain;
		var startTime = self.context.currentTime;
		gain.setValueAtTime(gain.value, startTime);
		var endTime = startTime + duration;
		gain.linearRampToValueAtTime(1, endTime);
	}
},

AudioLoader.prototype.fadeOutBGM = function(duration) {
	var self = this;
	if(self.playing_bgm_gain) {
		var gain = self.playing_bgm_gain.gain;
		var startTime = self.context.currentTime;
		gain.setValueAtTime(gain.value, startTime);
		var endTime = startTime + duration;
		gain.linearRampToValueAtTime(0, endTime);
	}
};
```

ゲーム内から、`AudioLoader` インスタンスの `fadeInBGM` あるいは `fadeOutBGM` を呼び出すことにより、
現在再生中のBGMをフェードイン／フェードアウトさせることができます。

`fadeInBGM` `fadeOutBGM` メソッドについて少しずつ解説していきたいと思います。

```
gain.setValueAtTime(gain.value, startTime);
```

現在時刻における音量を、現在の音量に設定しています。
設定しなくてもデフォルトの値(=現在の音量)になるのですが、
古いブラウザでは、デフォルトの値が 0 になっているブラウザもあるため、念のため設定します。

```
gain.linearRampToValueAtTime(1, endTime); // fade in
gain.linearRampToValueAtTime(0, endTime); // fade out
```

第一引数は最終的な音量です。第二引数に、いつまでにその最終的な音量まで減らす／増やすのかを設定します。
フェードインであれば、最終的な値は 1 になるので、第一引数に 1 を、フェードアウトならば、最終的な値は 0 になるので、
第一引数に 0 を指定しています。

