# 当たり判定

先述の移動の項目にて、自機・自機の弾・敵の移動を実装しました。
それでは、本項目では、当たり判定を実装し、以下を実現したいと思います。

* 自機と敵が接触すると、自機が死ぬ
* 自機の弾と敵が接触すると、敵が死ぬ

当たり判定について、キャラや敵は複雑な形をしており、厳密に接触しているかどうかを判定すると、
非常に計算に時間がかかるため、キャラや敵を矩形(くけい、と読みます。四角形のこと)扱いして、判定する方法と、
円扱いして判定する方法の大まかに2種類があります。

## 円同士の判定
```
ObjectBase.prototype.collisionRadius = function() {
	return 0;
};
ObjectBase.prototype.checkCollisionByCircle = function(obj) {
	if((this.x - obj.x)**2 * (this.y - obj.y)**2  <= (this.collisionRadius() + obj.collisionRadius())**2) {
		return true;
	}

	return false;
};
```

`collisionRadius` は当たり判定に使用する円の半径です。`checkCollisionByCircle` を用いて当たり判定を行います。
例えば、`chara.checkCollisionByCircle(enemy)` を実行することで、自機と敵の当たり判定を行うことができます。

`checkCollisionByCircle` メソッドの中身を見ていきましょう。円と円の当たり判定については、
2つの円の中心点同士の距離が、2つの円の半径の合計より短ければ衝突しています。

円1: 中心点(x1, y1) 半径r1
円2: 中心点(x2, y2) 半径r2

2つの円の中心点同士の距離: √(x1-x2)^2 + (y1-y2)^2
2つの円の半径の合計: r1 + r2

√(x1-x2)^2 + (y1-y2)^2 ≦ r1 + r2
であるならば衝突している。

JavaScript のコードにすると以下のようになります。

```
if(Math.sqrt((x1 - x2)**2 + (y1 - y2)**2) <= r1 + r2) {
	return true;
}
```

ただし、sqrt によるルートの計算は計算コストが高いので、同義の条件のまま、二乗した値を使うことが多いです。
```
if((x1 - x2)**2 + (y1 - y2)**2 <= (r1 + r2)**2) {
	return true;
}
```

`checkCollisionByCircle` メソッドは上記のコードを実装しています。

## 矩形同士の判定


```
ObjectBase.prototype.collisionWidth = function() {
	return 0;
};
ObjectBase.prototype.collisionHeight = function() {
	return 0;
};
ObjectBase.prototype.checkCollisionByRect = function(obj) {
	if(Math.abs(this.x - obj.x) < this.collisionWidth()/2 + obj.collisionWidth()/2 &&
		Math.abs(this.y - obj.y) < this.collisionHeight()/2 + obj.collisionHeight()/2) {
		return true;
	}

	return false;
};
```

矩形と矩形の衝突は、円と円の判定より多少面倒になります。
さらに、回転した正方形と回転した正方形の計算となるとさらに面倒になります。

今回は、正方形が回転していることは考えずに、正方形の辺がX軸、Y軸に常に平行である場合を考えたいと思います。
x, y それぞれについて、2つの矩形の距離が、2つの矩形の幅の合計より短ければ衝突しています。

矩形1: 中心点(x1, y1) 横幅 w1 縦幅 h1
矩形2: 中心点(x2, y2) 横幅 w2 縦幅 h2

2つの矩形の中心点同士の距離(x座標): (x1-x2)の絶対値
2つの矩形の中心点同士の距離(y座標): (y1-y2)の絶対値
2つの矩形の横幅の合計: w1/2 + w2/2
2つの矩形の縦幅の合計: h1/2 + h2/2

(x1-x2)の絶対値 ≦ w1/2 + w2/2
かつ
(y1-y2)の絶対値 ≦ h1/2 + h2/2

ならば衝突している。

JavaScript のコードにすると以下のようになります。
```
if(Math.abs(x1 - x2) <= w1/2 + w2/2 && Math.abs(y1 - y2) <= h1/2 + h2/2) {
	return true;
}
```

`checkCollisionByRect` メソッドは上記のコードを実装しています。`Rect` は `Rectangle` の略称で、矩形(四角形)の英単語です。
