# Signate Offroad Segmentation

URL: https://signate.jp/competitions/101 (認識制度部門)



### 実行手順

- google colaboratoryを使用
- 2~4のファイルに関しては、ファイル名の_foldXを0~5に変更して学習させる



### ソリューション概要(フォーラムのコピペ)

--------------------------------------------------------------------------------------------------------------------------------------

運営・ホストの皆様、今回はこのようなコンペを開催してくださり、ありがとうございました。

データ量もちょうどよく、初めてのセグメンテーションコンペでしたが、楽しく取り組むことができました。

個人の結果は残念でしたが、せっかくソリューション公開OKなコンペなので供養の意味もこめて公開します。

順位に関わらず、他の参加者の皆様の解法も共有頂けると嬉しいです。

(一旦日本語オンリーで失礼します。気が向いたら英訳します。)



Model

-  ① Unet, backborn resnet34, Focal Loss, 5fold (public: 0.8849235, private: 0.8684270)
  - output 3channel, 全データ
- ② Unet, backborn resnet18, Focal Loss, 5fold (public: 0.8840996, private: 0.8681062)
  - output 3channel, 全データ
- ③ Unet, backborn resnet34, BCE Loss, 5fold (public: 0.8828911, private: 0.8628499)
  - output 1channel: road, dirt road, othrt obstacleを別々のモデルで学習
  - other obstacleは全データで学習
  - efficientnet-b0でroadとdirt roadのどちらの面積が大きいかを二値分類 
    - road > dirt road, road < dirt road の画像のみを用いてそれぞれ学習
    - 計25個のUnetを使用

①, ②, ③を0.2, 0.2, 0.1の重みでavarage (public: 0.8861637, private: 0.8694099)



Validation set

- フォーラムでも議論されていた通りtrainやtestにはかなり類似している画像が含まれており、imagehash, networkxを組み合わせて類似している複数画像をgroup化しました。
- 参考:
  - https://www.kaggle.com/appian/panda-imagehash-to-detect-duplicate-images
  - https://www.kaggle.com/yukkyo/imagehash-to-detect-duplicate-images-and-grouping
- 類似画像をグループ化した後、以下のルールに沿ってバリデーションセットを作成しました。
  - train画像中の類似グループは、同じfoldに含めるものとする
    - validに類似画像が含まれると、CVの信頼性が下がる(?)
  - train, test両方に存在するグループは、学習時のすべてのfoldに含める
    - セグメンテーションタスクなので、類似しているからと言ってもそのまま教師データが使えるわけではない
    - すべてのfoldに含めることで、このグループには強い学習機を作れるはず(?)
- 後述しますが、これらは裏目に出てしまい、普通にrandam 5 foldの方が良かったかもしれません。



augmentation

- ResizeMixというのをtwitterで知って試してみて、Cutmixよりもスコアがよかったので採用しました。
- 一つ一つの領域が大きく、境界線を上手く取り出すことが重要なコンペだったので、直感的にもCutmixよりはResizeMixの方がよい気がしています。
- なぜかmixupは試しませんでした。試すくらいはしておくべきだった…。



Loss

- 序盤はずっとBCELossを使っていましたが、FocalLossに変えてからスコアがかなり上昇しました。



試したけど効かなかった

- pseudo labeling
- road or dirt roadの面積が大きい方を二値分類後、3channelで学習
  - BCELossでは若干スコアが良くなりましたが、FocalLossに変えてからはワークしませんでした。
- train Bを用いた手法
  - Perzさんのtrain B maskを使い、Bを学習後にAで学習、AとBを混ぜて学習、と両方試しましたがスコアは上がりませんでした。ここで打ち切ってしまって、Bを使うのは諦めてしまいました。
  - Bで2値にセグメンテーションするタスクをpretrainするというのはあったかもしれません。

考えられるshake downの要因

- 別フォーラムのコメントにも書きましたが、imagehashを使ったバリデーションセットの作成が裏目に出た可能性があります。「publicとprivateの割合も非公表だし、差が無いようにしてくれているだろう」という謎の勝手読みをしてしまい、LBをあげるのにこだわっていたのが一番の敗因で、もっと様々な可能性を考えるべきでした。
- Imagesizeをどのように決定したらよいかわからず、結局元々のサイズに近い(1056, 1920)を最後まで採用していました。ここのチューニングや、様々な画像サイズでのアンサンブルも試すべきでした。(Focal Lossが効くのに気付いたのが最終盤で、時間的に間に合いませんでした…) 
  このあたりの知見についてアドバイス頂けると嬉しいです。