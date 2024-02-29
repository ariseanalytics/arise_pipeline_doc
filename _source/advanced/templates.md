# 便利なテンプレートの紹介

ARISE-PIPIELINEでは、IFが明確に定められている各種Params（`ModelPipelineParams`、`PostFuncParams`等）を定義することで、パイプラインを構築する。`KFold`によるクロスバリデーションを実行する`TrainValSplitParams`など、良く用いられることが想定されるParamsについてはそれらを簡単に生成可能なテンプレートを`arise_pipeline.templates`以下に用意している。ここではその一部を紹介する。また、各プロジェクトで構築されたParamsのうち、汎用的なものをテンプレート化していくことによって、プロジェクト横断で共通のモジュールを構築していくことを目指していきたいので、各チームで構築したものはぜひ運営にも共有していただきたい。


## クロスバリデーションの実施

### できること
- `KFold`, `GroupKFold`, `StratifiedKFold`によるクロスバリデーションの実施


### 使用するメソッド等
- モジュール：`arise_pipeline.templates.ml_pipeline.sub_params.train_val_split`
- メソッド：`create_stratifiedkfold_split_params`,
- 生成対象Params：`TrainValSplitParams`

### 使い方の紹介（StratifiedKFoldの例）

今回は`StratifiedKFold`を例にテンプレートの使い方を紹介する。

まず、`StratifiedKFold`は、クロスバリデーションの手法の一つで、クラスの分布を保ったまま各foldを作成するものである。つまり、各fold内のクラスの割合が元のデータセットと近い状態で分割され、各foldごとの条件が近しくなる特徴がある。

ARISE-PIPELINEでは、`StratifiedKFold`によるクロスバリデーションを可能とする`TrainValSplitParams`を生成するテンプレートメソッドを用意しているため、こちらの使い方を説明する。


#### 1. テンプレートメソッドによる`TrainValSplitParams`の生成（pythonファイル）

まず、`StratifiedKFold`用の`TrainValSplitParams`を生成する必要があるが、これは次のメソッドにより簡単に生成できる。

```python
from arise_pipeline.templates.ml_pipeline.sub_params.train_val_split import create_stratifiedkfold_split_params


skf_params = create_stratifiedkfold_split_params()
```

以上で生成したインスタンスについて、以下の疑似コードのように`ModelPipelineParams`の引数に代入してあげればよい。

```python
from arise_pipeline.ml_pipeline.main_params import ModelPipelineParams

model_pipeline_params = ModelPipelineParams(
    ...,
    train_val_split_params=skf_params
)
```

あとは、他のパラメータを適切に用意し、`MlPipeline`クラスに渡してあげればよく、これ以上の説明は割愛する。

#### 2. パラメータファイルへの必要事項の記載

上記はコードに関する説明だが、これに加えてパラメータファイルに必要なパラメータを記載する必要がある。本テンプレートでは内部では、以下の関数を`TrainValSplitParams`の`func`に渡している。

```python

import pyspark.sql.dataframe


def stratifiedkfold_split(
    data: pyspark.sql.dataframe.DataFrame,
    numFolds: int,
    labelCol: str,
    seed: int | None
) -> pyspark.sql.dataframe.DataFrame:
    skfold = StratifiedKFold(numFolds=numFolds, outputCol="fold", labelCol=labelCol, seed=seed)
    return skfold.transform(data)
```

上記の関数の引数のうち`data`以外は、同名のパラメータをパラメータファイルから参照する仕組みとなっているため、次の3つのパラメータをパラメータファイルに記載する必要がある。

```yaml
numFolds: 5, # fold数
labelCol: "label", # StratifiedKFoldで参照するラベルが格納されているカラム名
seed: 42, # seed値
```

以上のセットアップにより、`StratifiedKFold`によるクロスバリデーションが利用可能となる。


## 特徴量重要度算出

### できること
- SynapseMLのLightGBMの特徴量重要度を算出し、csvとして吐き出す


### 使用するメソッド等
- モジュールパス：`arise_pipeline.templates.ml_pipeline.sub_params.post_func`
- メソッド：`create_extract_lightgbm_feature_importances_func_params`,
- 生成対象Params：`PostFuncParams`

### 使い方の紹介

#### 0. 前提条件

まず、こちらを利用するに際して、以下の条件を満たす`ModelPipelineParams`を設定してある必要がある。

【前提条件】
- `ModelPipelineParams`中の`model_params`に与える`build_func`については以下2つの処理を内包したPipelineを生成する関数となっていること
    - `pyspark.ml.feature.VectorAssembler`による特徴量のベクトル化処理
    - `synapse.ml.lightgbm`モジュールのLightGBM

例えば、以下のようなものであればよい。（各引数についてはパラメータファイルに記載してある前提）

```python
from typing import Any

from pyspark.ml.feature import VectorAssembler
import pyspark.ml.Pipeline
from synapse.ml.lightgbm import LightGBMClassifier

from arise_pipeline.ml_pipeline.sub_params import PysparkPipelineParams


def build_lightgbm_pipeline(
    featureCols: str,
    featuresModelInputCols: str,
    labelCol: str,
    predictionCol: str,
    lgbm_params: dict[str, Any]
) -> pyspark.ml.Pipeline:
    va = VectorAssembler(inputCols=featuresCols, outputCol=featuresModelInputCols)
    lgbm = LightGBMClassifier(
        objective="multiclass",
        featureCol=featuresModelInputCols,
        labelCol=labelCol,
        predictionCol=predictionCol,
        **lgbm_params
    )
    pipeline = pyspark.ml.Pipeline(stages=[va, lgbm])
    return pipeline


main_params = PysparkPipelineParams(
    name="lightgbm_pipeline",
    build_func=build_lightgbm_pipeline
)
```

#### 1. テンプレートメソッドによる`PostFuncParams`の生成（pythonファイル）

先に述べた前提条件を守った上で、特徴量重要度を算出する`PostFuncParams`を生成する。使い方としては以下のように`create_extract_lightgbm_feature_importances_func_params`に一定の引数を渡してあげるだけでよい。

```python
from arise_pipeline.templates.ml_pipeline.sub_params.post_func import create_extract_lightgbm_feature_importances_func_params


pipeline_ds_name = "lightgbm_pipeline"
output_ds_name = "lgbm_feature_importences"
filename = "feature_importances.csv"
tags = ["train"]

lgbm_fi_extract_params = create_extract_lightgbm_feature_importances_func_params(
    pipeline_ds_name=pipeline_ds_name,
    output_ds_name=output_ds_name,
    filename=filename,
    tags=tags,
)
```

ここで各引数については、以下の説明の通りである。

- `pipeline_ds_name`は、特徴量重要度を算出する対象のLightGBMが入っているパイプラインのデータセット名である。これはすなわち、`ModelPipelineParams`中の`model_params`に与える`name`と同一である。
- `output_ds_name`はカタログに登録されるアウトプットのデータセット名
- `filename`は保存されるファイル名で、出力先は08.reporting
- `tags`は任意の文字列（のリスト）を設定することができるがが、ここで指定したタグを`MlPipeline`の`make`コマンドの`tags`に指定した時のみ、こちらが実行される。

上記で作成した`lgbm_fi_extract_params`を以下の疑似コードのように`PostPipelineParams`に渡してあげれば良い。

```python
from arise_pipeline.ml_pipeline.main_params import PostPipelineParams


post_pipeline_params = PostPipelineParams(
    ...,
    post_funcs=[lgbm_fi_extract_params],
)
```

#### 2. パラメータファイルへの必要事項の記載

パラメータファイルに記載すべき内容と例は以下の通りである。

```yaml
importance_type: "gain", # 特徴量重要度の種類。"split" or "gain"。
feature_importances_col: "feature_importance", # 算出した特徴量重要度を格納するカラム名（アウトプットのpd.DataFrame）
features_name_col: "feature_name", # 特徴量名を格納するカラム名（アウトプットのpd.DataFrame）
```

以上の例に従いパラメータファイルの記載を完了すれば、セットアップ完了である。

### Appendix

上記のアウトプットを利用して、特徴量重要度のプロットを作成する`PostFuncParams`のテンプレートもある。使い方としては他のテンプレートと変わらないので、説明は割愛するが、こちらを利用することで`plotly`ベースでグラフを描画できるので、興味がある方はぜひ利用していただければと。

- モジュールパス：`arise_pipeline.templates.ml_pipeline.sub_params.post_func`
- メソッド：`create_feature_importances_plot`,
- 生成対象Params：`PostFuncParams`