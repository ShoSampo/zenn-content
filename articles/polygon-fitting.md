---
title: "Turning Functionとアフィン変換を用いた地図ポリゴンのフィッティング"
emoji: "📐"
type: "tech"
topics: ["アルゴリズム", "Python", "幾何学", 形状分析, "ComputerVision"]
published: true
published_at: 2024-12-20
---

# 2. アフィン変換によるポリゴンマッチング
## 2.1 特徴点の抽出と評価

### 特徴点抽出の目的
ポリゴンの位置合わせにおいて、形状を大きく特徴づける角（特徴点）を抽出することは重要です。これは以下の理由によります：

1. **形状の大域的な特徴の捕捉**
   - ポリゴンの全体的な形状を決定づける主要な角を特定
   - 細かな凹凸ではなく、形状の本質的な特徴を表現
   - アフィン変換のための信頼性の高い対応点として機能

2. **効率的なマッチング**
   - 全頂点ではなく、重要な特徴点のみを使用
   - 計算コストの削減
   - ノイズに対するロバスト性の向上

### 特徴点抽出手法の選択
当初、Turning Functionを用いた角検出を検討しましたが、ノイズの含まれる実際のポリゴンデータでは期待通りの結果が得られませんでした。
一方で、画像認識の分野で実績のあるShi-Tomasiのコーナー検出は、ノイズにロバストで、より効果的な結果を示しました。

#### Turning Functionの課題
- 小さな凹凸やノイズに敏感に反応
- ポリゴンの大域的な角の特定が困難
- 形状を特徴づける角とノイズの区別が難しい

※ 以下のTurning Functionにおいて、データ点が突出している箇所は、ポリゴンにおける小さな凹凸部分と対応します。

![](/images/turning-function/fig-9.png)

#### 画像認識ベースの手法を選択した理由
- Shi-Tomasiアルゴリズムは小さな凹凸に影響されにくい
- OpenCVの十分に検証された実装が利用可能
- 形状を特徴づける角を効果的に抽出できる

![](/images/turning-function/fig-8.png)

### アルゴリズムの詳細
本実装では、Shi-Tomasiのコーナー検出アルゴリズムを採用しています。このアルゴリズムは、Harris Corner Detectorを改良して角検出に特化させたもので、より安定したコーナー検出が可能です。

アルゴリズムの詳細な解説については以下を参照してください：
- [OpenCV公式チュートリアル（Harrisコーナー検出）](https://labs.eecs.tottori-u.ac.jp/sd/Member/oyamada/OpenCV/html/py_tutorials/py_feature2d/py_features_harris/py_features_harris.html)
- [OpenCV公式チュートリアル（Shi-Tomasi）](https://labs.eecs.tottori-u.ac.jp/sd/Member/oyamada/OpenCV/html/py_tutorials/py_feature2d/py_shi_tomasi/py_shi_tomasi.html)
- [Harris Corner Detectionの解説](https://biocv.hateblo.jp/entry/2024/02/11/211232)

#### OpenCVによる実装
OpenCVの`cv2.goodFeaturesToTrack()`メソッドは、デフォルトでShi-Tomasiのコーナー検出アルゴリズムを使用します。このメソッドの詳細なパラメータや使用方法については、[OpenCV公式ドキュメント](https://docs.opencv.org/3.4/dd/d1a/group__imgproc__feature.html#ga1d6bb77486c8f92d79c8793ad995d541)を参照してください。

#### 特徴点検出の実装例
```python
def detect_corners(image: np.ndarray) -> np.ndarray:
    """
    画像からコーナーを検出する関数

    Args:
        image: 入力画像

    Returns:
        np.ndarray: コーナーの座標の配列
    """
    return cv2.goodFeaturesToTrack(image, maxCorners=25, qualityLevel=0.14, minDistance=50)

def find_characteristic_corners(polygon: Polygon) -> List[int]:
    """
    ポリゴンの特徴点を見つける関数
    
    Args:
        polygon (Polygon): 対象のポリゴン
    
    Returns:
        List[int]: 特徴点の頂点インデックスのリスト
    """    
    image_size = (1000, 1000)
    margin_ratio = 0.1
    # ポリゴンを画像に変換
    img, scale_x, scale_y, x_min, y_min = polygon_to_image(
        polygon, image_size, margin_ratio
    )
    
    # コーナー検出
    image_corners = detect_corners(img)
    
    # 画像座標系から元の座標系（緯度経度）に変換
    geo_corners = image_to_real_coordinates(
        image_corners, image_size, margin_ratio,
        scale_x, scale_y, x_min, y_min
    )
    
    # 最も近いポリゴンの頂点のインデックスを取得
    polygon_coords = list(polygon.exterior.coords)[:-1]
    return find_nearest_vertex_indices(geo_corners, polygon_coords)
```
### 特徴点のマッチング
抽出された特徴点は、Turning Functionの特性を活用してマッチングを行います：

1. **局所的なTurning Function**
   - 各特徴点周辺のTurning Functionパターンを計算
   - 特徴点での角度変化の特徴を捉える

2. **マッチング基準**
   - 特徴点周辺のTurning Function形状の類似度
   - 特徴点間の相対的な位置関係
   - 大域的な形状整合性の考慮

このように抽出された特徴点とそのマッチングは、後続のアフィン変換の精度と安定性を大きく左右する重要な要素となります。

## 2.2 アフィン変換の適用
- 変換パラメータの推定
- 変換の適用と結果の評価
- 処理における制限事項

# まとめ
- 手法の特徴と利点
- 適用可能な場面と注意点
- 今後の課題
