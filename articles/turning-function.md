---
title: "Turning Functionを用いたポリゴン形状マッチング"
emoji: "🧩"
type: "tech"
topics: ["アルゴリズム", "Python", "幾何学", "形状分析", "ComputerVision"]
published: true
published_at: 2024-12-03
---
![](/images/header-toggle.png)

こんにちは、トグルホールディングスのAIエンジニアの[中村](https://note.com/toggle/n/nbb9e9d9ee328)です！

[トグルホールディングスエンジニアアドベントカレンダー](https://engineer-blog.toggle.co.jp/)の3日目の記事です！ 

業務で地図ポリゴンデータの形状分析をする機会があったので、その際に用いた手法を紹介します！

# はじめに

形状の類似性を評価する際、多くの場合、データを画像に変換してから画像認識の手法（SIFT/SURFなどの特徴点検出や輪郭抽出など）を適用します。しかし、元々座標データとして存在するポリゴンの場合、画像変換を介さない直接的なアプローチの方が効率的です。

本記事では、ポリゴンの座標データをそのまま分析できるTurning Functionを紹介します。この手法は、ポリゴンの境界線に沿った角度変化とエッジの長さをステップ関数として表現することで、形状の類似性を評価することができます。特に以下のような場面で有用です：

1. **大量のポリゴンデータを効率的に処理したい場合**
   - 画像変換のオーバーヘッドを避けられる
   - メモリ使用量を抑えられる

2. **座標の精度を保ったまま形状を比較したい場合**
   - 画像化による情報の損失がない
   - 元の座標データの精度を維持

3. **シンプルで保守性の高い実装を目指す場合**
   - 画像処理ライブラリへの依存がない
   - 純粋な幾何計算のみで実現

# 1. Turning Functionの概要
Turning Function（回転関数）は、ポリゴンの形状を1次元のステップ関数として表現する手法です。ポリゴンの境界に沿って反時計回りに進みながら、各頂点での接線の角度変化を累積的に記録していきます。

![](/images/turning-function/fig-1.png)

## 1.1 基本原理
1. **境界の走査**
   - ポリゴンの境界を反時計回りに走査
   - 開始点から終了点まで、境界に沿って移動

2. **関数の軸**
   - 横軸：境界上の位置（エッジの長さを外周長で正規化：[0,1]）
   - 縦軸：累積角度（反時計回りを正として累積）
      - 最終的な累積角度は2π（360度）に到達

3. **角度の累積**
   - 各頂点で内角に基づく角度変化を記録
   - ステップ関数として表現される
   - 頂点間（エッジ上）では角度は一定

## 1.2 特徴-不変性

Turning Functionの重要な特徴は、以下の変換に対して不変であることです：

1. **回転に対する不変性**
   - ポリゴンを回転させても同じ関数が得られる
   - 向きの異なるポリゴンの比較が可能

2. **平行移動に対する不変性**
   - 位置が異なっても同じ関数が得られる
   - 絶対座標に依存しない比較が可能

3. **スケールに対する不変性**
   - サイズの異なるポリゴンでも比較可能
   - 正規化により相対的な形状を評価

以下のプロットのように、回転、平行移動、スケール変換に対して不変であるため、形状が同じ２つのポリゴンのTurning Functionがぴったりと重なっていることがわかります。

![](/images/turning-function/fig-4.png)

## 1.3 実装例
Turning Functionの主要な部分を以下のようなPythonクラスで実装しました。
ここでは、核となる機能のみを示します（完全な実装ではないことに注意してください）：

```python
class TurningFunction(BaseModel):
    """
    ポリゴンのTurning Functionを表現するクラス。
    弧長と回転角の配列を保持し、形状の非類似度計算などに使用する。
    """
    arc_lengths: NDArray[np.float64] = Field(
        description="累積弧長の配列"
    )
    turning_angles: NDArray[np.float64] = Field(
        description="累積回転角の配列"
    )

    model_config = {
        "frozen": True,
        "arbitrary_types_allowed": True
    }

    @classmethod
    def from_polygon(cls, polygon: Polygon) -> TurningFunction:
        """
        ポリゴンからTurning Functionのインスタンスを生成

        Args:
            polygon (Polygon): 入力ポリゴン

        Returns:
            TurningFunction: 生成されたTurningFunction
        """
        coords = get_polygon_coordinates(polygon)
        arc_lengths = cls._calculate_arc_lengths(coords)
        turning_angles = cls._calculate_turning_angles(coords)
        return cls(arc_lengths=arc_lengths, turning_angles=turning_angles)

    def normalize(self) -> TurningFunction:
        """
        弧長を正規化したTurning Functionを返す

        Returns:
            TurningFunction: 正規化されたTurning Function
        """
        total_length = self.arc_lengths[-1]
        normalized_lengths = self.arc_lengths / total_length
        return TurningFunction(
            arc_lengths=normalized_lengths,
            turning_angles=self.turning_angles
        )

    @staticmethod
    def _calculate_arc_lengths(coords: List[Coordinate]) -> NDArray[np.float64]:
        """座標列から弧長を計算"""
        distances = np.array([
            TurningFunction._calculate_distance(p1, p2) 
            for p1, p2 in zip(coords, coords[1:])
        ])
        return TurningFunction._prepend_zero(np.cumsum(distances))

    @staticmethod
    def _calculate_turning_angles(coords: List[Coordinate]) -> NDArray[np.float64]:
        """座標列から回転角を計算"""
        turning_angles = np.array([
            TurningFunction._calculate_angle(p1, p2, p3) 
            for p1, p2, p3 in zip(
                coords, 
                coords[1:], 
                coords[2:] + [coords[1], coords[2]]
            )
        ])
        return TurningFunction._prepend_zero(np.cumsum(turning_angles))

    @staticmethod
    def _calculate_angle(p1: Coordinate, p2: Coordinate, p3: Coordinate) -> float:
        """3点の座標を受け取り、それらの点を結ぶベクトルの角度を計算して返す"""
        v1 = p2.to_numpy() - p1.to_numpy()
        v2 = p3.to_numpy() - p2.to_numpy()
        
        v1_u = v1 / np.linalg.norm(v1)
        v2_u = v2 / np.linalg.norm(v2)
        
        dot_product = np.dot(v1_u, v2_u)
        cross_product = v1_u[0] * v2_u[1] - v1_u[1] * v2_u[0]
        
        return np.arctan2(cross_product, dot_product)

    @staticmethod
    def _calculate_distance(p1: Coordinate, p2: Coordinate) -> float:
        """2点間のユークリッド距離を計算する"""
        return np.linalg.norm(p2.to_numpy() - p1.to_numpy())

    @staticmethod
    def _prepend_zero(values: NDArray[np.float64]) -> NDArray[np.float64]:
        """配列の先頭に0を追加する"""
        return np.insert(values, 0, 0)
```

このクラスは以下の特徴を持ちます：

1. **データの保持**
   - 累積弧長（arc_lengths）
   - 累積回転角（turning_angles）

2. **主要な機能**
   - ポリゴンを受け取りインスタンス生成（from_polygon）
   - 弧長の正規化（normalize）
   - 回転角度計算（_calculate_turning_angles）
   - 弧長計算（_calculate_arc_lengths）


### 使用例
以下は、正方形のポリゴンに対してTurning Functionを計算する簡単な例です：

```python
from shapely.geometry import Polygon

# 正方形のポリゴンを用意
square_polygon = Polygon([
    (0, 0), 
    (1, 0), 
    (1, 1), 
    (0, 1), 
    (0, 0)  
])

# Turning Functionを計算
square_tf = TurningFunction.from_polygon(square_polygon)
# 正規化
square_tf = square_tf.normalize()

print(square_tf)
# 出力:
# TurningFunction(
#     arc_lengths=array([0.  , 0.25, 0.5 , 0.75, 1.  ]), 
#     turning_angles=array([0.        , 1.57079633, 3.14159265, 4.71238898, 6.28318531])
# )
```

以下に様々な形状のポリゴンのTurning Functionをプロットした具体例を示します。

## 1.4 様々な形状での具体例

**単純な正方形の場合：**

- 各頂点で90度（π/2）ずつ増加
- 4つの頂点で合計360度（2π）
- エッジの長さは全て等しいため、横軸上で等間隔にステップが現れる

![](/images/turning-function/fig-2.png)

**円に近い多角形の場合：**

- 各頂点での角度変化とエッジの長さが小さいため、ほぼ直線となる

![](/images/turning-function/fig-5.png)

**星型の場合：**

- 角度の変化が多いため、ステップ関数の形状が複雑になる
- 角度の変化が負になる（時計回り）頂点が存在するため、ステップ関数が下る箇所が存在する

![](/images/turning-function/fig-3.png)


## 1.5 形状の数値化がもたらす利点

1. **定量的な比較が可能**
   - 形状の類似度を数値として評価
   - 大量のポリゴンの比較に適用可能

2. **計算効率**
   - 2次元の形状を弧長と角度の関係を表す関数として表現
   - 形状の比較を関数の比較として扱える
     - 後述するL2ノルムを用いた類似度計算が可能



このような特徴により、Turning Functionはポリゴンデータの形状マッチングに適した手法となっています。

# 2. L2ノルムによる形状の類似度評価

2つのポリゴンA, Bの形状の類似度は、それぞれのTurning Function $T_A(l)$, $T_B(l)$ のL2ノルムを用いて計算できます。

## 2.1 数式による定義
類似度（より正確には非類似度）$S(A,B)$は以下の式で定義されます：

$$S(A,B) = d(A,B) = \|T_A - T_B\|_2 = \left(\int_0^1(T_A(l) - T_B(l))^2 dl\right)^{\frac{1}{2}}$$

ここで：
- $l$は正規化された累積長さ（0から1の範囲）
- $T_A(l)$, $T_B(l)$はそれぞれのポリゴンのTurning Function
- $S(A,B)$は2つのポリゴン間の非類似度を表す

## 2.2 特徴
1. **非類似度としての解釈**
   - $S(A,B)$の値が小さいほど、2つのポリゴンの形状は類似
   - 完全に同一の形状の場合、$S(A,B) = 0$

2. **対称性**
   - $S(A,B) = S(B,A)$が成り立つ
   - どちらのポリゴンを基にしても同じ結果

3. **正規化の効果**
   - スケールに依存しない比較が可能
   - 回転に対して不変

この類似度計算により、形状の定量的な比較が可能となり、最も類似したポリゴンの組み合わせを自動的に特定することができます。

## 2.3 類似度評価の具体例

**完全な同一形状のポリゴンの場合：**

- 非類似度が0となる
  - L2ノルム：0.00

![](/images/turning-function/fig-4.png)

**軽微な差がある類似形状の場合：**

- 非類似度が0に近い値となる
  - L2ノルム：0.35

![](/images/turning-function/fig-7.png)

**形状が異なる場合：**

- 非類似度が比較的大きい値となる
  - L2ノルム：1.09
![](/images/turning-function/fig-6.png)


# 3. ポリゴンマッチング
## 3.1 マッチングの基本原理
形状は似ているが、頂点座標が完全には一致しないポリゴン同士のマッチングを考えます。例えば、以下の図のように、元のポリゴン群（青）に対して、頂点位置にノイズを加えたポリゴン群（赤）があり、それぞれ対応する最適なペアを見つけ出す問題を考えます：

![](/images/turning-function/fig-8.png)

このようなマッチング問題は、以下のステップで解決できます：

1. **非類似度行列の作成**
   - 全てのポリゴンペアの組み合わせについてTurning Functionの非類似度（L2ノルム）を計算
   - N×Mの非類似度行列を生成（Nは群Aのポリゴン数、Mは群Bのポリゴン数）

2. **最適なペアの選択**
   - 非類似度に基づいて最適なペアを決定
   - 同じ形状のポリゴン同士で最小の非類似度を示すことが期待される

## 3.2 マッチング結果の評価

### 非類似度行列の解釈
以下の表は、各ポリゴンペアの非類似度（L2ノルム）を示しています。対角線上の太字の値は、同じ形状のノイズ有無のポリゴン同士の非類似度を表しており、これらが行ごとの最小値となっていることがわかります：

| ポリゴンの形状 | 平行四辺形 | 五角形 | 矢印 | 稲妻 | 星型 | K字 |
|-|-|-|-|-|-|-|
| **ノイズ付き平行四辺形** | **0.0414** | 0.2938 | 0.9633 | 1.0204 | 1.1828 | 1.7331 |
| **ノイズ付き五角形** | 0.4143 | **0.1202** | 1.2547 | 1.0089 | 1.3688 | 1.5267 |
| **ノイズ付き矢印** | 0.7837 | 0.9969 | **0.3899** | 1.3512 | 1.0961 | 2.0465 |
| **ノイズ付き稲妻** | 1.1630 | 1.1155 | 1.6363 | **0.2909** | 1.4559 | 2.2787 |
| **ノイズ付き星型** | 1.2671 | 1.4768 | 0.9425 | 1.5043 | **0.4368** | 2.7275 |
| **ノイズ付きK字** | 1.5091 | 1.3319 | 2.0238 | 1.8337 | 2.2843 | **0.5794** |


### マッチング結果の可視化
以下の図は、各ペアのポリゴン形状（左）とそのTurning Function（右）を示しています。非類似度が最小となったペアでは、ノイズの影響はあるものの、Turning Functionが良く一致していることがわかります：

![](/images/turning-function/fig-9.png)
![](/images/turning-function/fig-10.png)
![](/images/turning-function/fig-12.png)
![](/images/turning-function/fig-11.png)
![](/images/turning-function/fig-13.png)
![](/images/turning-function/fig-14.png)

### ポリゴンマッチングにおける重要な注意点
Turning Functionを用いたポリゴンマッチングでは、始点の位置合わせが特に重要です。これは、Turning Functionが始点から反時計回りに累積角度を記録していく性質上、始点の選び方によって関数の形状が大きく変化するためです。

**始点の位置合わせ**
   - Turning Functionの計算には始点の位置が重要
   - 類似する2つのポリゴンでは、形状上の同じ位置を始点として設定する必要がある
   - 始点が異なると、類似形状でもL2ノルムが大きくなってしまう

例えば、同じ形状の2つのポリゴンであっても、一方は左上の頂点から、もう一方は右下の頂点からTurning Functionを計算すると、得られる関数は大きく異なります。そのため、マッチングの前処理として、比較するポリゴン同士で始点を適切に揃えることが、正確な類似度評価には不可欠です。

# まとめ

本記事では、ポリゴンの形状マッチングにおけるTurning Functionの活用について解説しました。主なポイントは以下の通りです：

1. **座標データの直接活用**
   - 画像変換を介さない効率的な形状分析が可能
   - 座標の精度を保ったまま比較が可能

2. **Turning Functionの特徴**
   - 回転・平行移動・スケールに対する不変性
   - 形状を1次元のステップ関数として表現
   - L2ノルムによる定量的な類似度評価

3. **実践的な応用**
   - ノイズを含むポリゴン同士のマッチング
   - 始点位置の適切な設定の重要性


後日、今回の記事の続編となる「Turning Functionとアフィン変換を用いた地図ポリゴンのフィッティング」にて、より実践的な地図ポリゴンのフィッティングについて解説します！お楽しみに！