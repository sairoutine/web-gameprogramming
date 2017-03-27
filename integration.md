# 環境構築
本書では、タスクランナーである gulp を使い、開発環境を構築したいと思います。
これは、下記の目的を満たしたいためです。

* クラス毎にファイルを分割したいため
* コード修正をブラウザですぐに確認したいため

前者の「クラス毎にファイルを分割したいため」については、
ゲームプログラムは他のプログラムより、クラス数が多くなりがちだからです。
1つの js ファイルに全てのクラスを記述したり、あるいは分割した js を
それぞれ `script` タグで一つ一つ読み込んでも良いのですが、コードの保守のしやすさのため、
クラス毎にファイルを分割することにし、それを `browserify` で結合することにします。
(`browserify` については後述します。)

後者の「コード修正をブラウザですぐに確認したいため」については、
プログラムを修正するたびに、ブラウザを開いてリロードボタンを押す手間を省くためです。
ゲーム開発は画面の配置調整など含めて、細かくコードを修正しては画面の確認を繰り返すことが多いです。
コードを修正するとブラウザを自動でリロードしてくれるタスクを導入して、この手間を減らします。

なお、以降の章では、特に `gulp` に依存した記述をしておりませんので、
自分にとって得意な開発環境があるを作れる方は、そちらを使用していただいても構いません。
例えば、`babel` であったり、`webpack` を使用して頂いても、以降の章では差し支えありません。

# ディレクトリ構成
ディレクトリ構成については下記のようにしたいと思います。
```
.
├── package.json
├── public
│   ├── bgm
│   ├── image
│   ├── js
│   │   ├── main.js
│   ├── index.html
│   └── sound
└── src
    └── js
        ├── asset_loader
        ├── logic
        ├── object
        ├── scene
        ├── game.js
        └── main.js
```

  **package.json**  
`npm init` により生成します。`npm` についての説明は省略します。

  **public/bgm**  
ゲームで使用する BGM ファイルをここに置くことにします。

  **public/image**  
ゲームで使用する画像ファイルをここに置くことにします。

  **public/sound**  
ゲームで使用するSEファイル(効果音)をここに置くことにします。

  **public/js**  
`gulp` 及び `browserify` でビルドされた js ファイルをここに置くことにします。
ゲームは、この js ファイルを使用します。`gulp` 及び `browserify` によって自動生成されるので、
このファイルを編集してはいけません。

  **public/index.html**  
ゲームを表示するためのHTMLファイルです。

  **src/js/logic**  
ゲームのソースコードのうち、画像や音声データの読み込みをするクラスをここに置くことにします。

  **src/js/logic**  
ゲームのソースコードのうち、ゲームロジックに値するクラスはここに置くことにします。

  **src/js/object**  
ゲームのソースコードのうち、オブジェクト(後述します)に値するクラスはここに置くことにします。

  **src/js/scene**  
ゲームのソースコードのうち、シーン(後述します)に値するクラスはここに置くことにします。

  **src/js/game.js**  
ゲームのソースコードのうち、ゲーム全体を表すクラスである Game クラスを記述するファイルです。

  **src/js/main.js**  
ゲームのソースコードのうち、Game クラスの生成及び実行をするファイルです。エントリポイントとも呼びます。

# gulp
`gulp` はタスクランナーです。色々なタスクを自動化するために使います。
今回は、以下のタスクを自動化するために使用します。

* クラス毎に分割したファイルを結合するため
* コードの修正をするとブラウザをすぐリロードする

上記の2つのタスクは `browserify` と `browser-sync` モジュールを使用して実現します。

`gulp` の watch 機能 を使用することで、コードの修正にフックして、
自動でファイル結合と、ブラウザリロードを行うことにします。

# browserify
`browserify` とは `require()` 関数を使うためのモジュールです。`require()` とは、あるソースコード内から、他のソースコードを読み込むための関数です。node を利用したことがある人は、既に `require` 関数が用意されているので、使用されたことがある人も多いかもしれません。`require()`関数を使うことで、クラス毎に適切に js ファイルを分割することができ、最終的にそれらは一つの js ファイルになって出力されます。

# browser-sync

`browser-sync` はファイル変更を監視し、自動でブラウザのリロードを行ってくれるツールです。
ローカルサーバーも起動してくれるので、ゲームの開発中は `browser-sync` を使用することで、
ゲームの動作確認の手間を大幅に減らすことができます。


# gulpfile の作成
それでは、上記のタスクを利用するために必要な npm モジュールをインストールします。
```
npm init
npm -i browser-sync browserify gulp gulp-notify gulp-plumber gulp-rename run-sequence vinyl-source-stream
```

下記のgulpfileを用意します。
```
'use strict';

// ソース元の対象ファイル
var source_file = './src/js/main.js';
// 出力ディレクトリ
var dist_dir = './public/js/';
// アプリファイル
var appjs = 'main.js';

// gulp watch で開く html
var html_dir = "public";
var html = "index.html";

var browserify = require('browserify');
var gulp       = require('gulp');
var source     = require('vinyl-source-stream');
var rename     = require('gulp-rename');
var plumber    = require('gulp-plumber');
var runSequence= require('run-sequence');
var path       = require('path');
var notify     = require('gulp-notify');
var browserSync= require('browser-sync').create();

gulp.task('browserify', function() {
	return browserify(source_file)
		.bundle()
		.on('error', function(err){   //ここからエラーだった時の記述
			// デスクトップ通知
			var error_handle = notify.onError('<%= error.message %>');
			error_handle(err);
			this.emit('end');
		})
		.pipe(source(appjs))
		.pipe(gulp.dest(dist_dir));
});

gulp.task('reload', function () {
	return browserSync.reload();
});

gulp.task('build', function(callback) {
	return runSequence(
		'browserify',
		'reload',
		callback
	);
});

gulp.task('browser-sync', function() {
	return browserSync.init({
		server: {
			baseDir: html_dir,
			index: html,
		}
	});
});

gulp.task('watch', ['browser-sync'], function() {
	gulp.watch('src/js/**/*.js', ['build']);
});
```

これで `gulp watch` をコマンドライン上で実行することで、gulp の監視タスクが立ち上がます。
ソースコードの変更があると、自動でファイルの結合とブラウザのリロードを行ってくれます。
