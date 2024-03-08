# Optunaを用いたパラメータチューニング

## はじめに

MLPipelineには、標準でOptunaによるパラメータチューニング機能を搭載している。ここではそちらの使い方を紹介する。

### できること
- モデルパイプラインに含まれる各種パラメータのチューニング
    - `ModelPipelineParams.main_model`
    - `ModelPipelineParams.preprocess_model`
    - `ModelPipelineParams.train_only_func`

### できないこと

- モデルパイプライン以外に含まれる各種パラメータのチューニング

※ただし、自前でOptunaのチューニングコードを書いて、`MlPipeline`自体を包む形でチューニングを行うことは可能


## 利用方法

### おおまかな流れ

まず、全体の大まかな流れは次の通りである。今回は2のみを説明する。

1. `ModelPipelineParams`以外の各種paramsの準備や設定ファイルの準備を行う
1. `ModelPipelineParams`中でチューニングコードや条件を記載する
    1. a
    1. b
    1. c
1. `MlPipeline`クラスのビルド・実行

### `ModelPipelineParams`の設定

基本的に使うのは

- `model_params`、`train_val_split_params`は必ず定義しておく
- `preprocess_params`、`train_only_func`は必要なものについては定義しておく
- `use_optuna = True`とする
- `optuna_params`に用意した`OptunaParams`を渡す

### `OptunaParams`の設定


- `optimize_params`
    `study.optimize`に引き渡すパラメータ
- `create_study_params`
    `create_study`に渡すパラメータ
- `score_func`

- `model_trial`
- `preprocess_trial`
- `train_only_func_trial`

#### `optimize_params`

```python
optimize_params = {
    "n_trials": 20
}
```

#### `create_study_params`

```python
create_study_params = {
    "direction": "maximize",
    "study_name": "iris_sample_tuning",
}

from arise_pipeline.ml_pipeline.sub_params import OptunaParams, PysparkPipelineParams
from arise_pipeline.ml_pipeline.main_params import ModelPipelineParams


OptunaParams(
    optimize_params=optimize_params,
    create_study_params=create_study_params,
    preprocess_trial=None,
    model_trial=build_model_trial,
    train_only_func_trial=None,
    score_func=score_func
)
```

#### `score_func`

```yaml

metricName: f1
```

```python
from pyspark.ml import Pipeline
from pyspark.ml.evaluation import MulticlassClassificationEvaluator
import pyspark.sql.dataframe

def score_func(
    evaluate_sdf: pyspark.sql.dataframe.DataFrame,
    labelModelInputCol: str,
    predictionCol: str,
    metricName: str,
) -> dict[str, float]:
    evaluator = MulticlassClassificationEvaluator(
        labelCol=labelModelInputCol,
        predictionCol=predictionCol,
        metricName=metricName,
    )
    score = evaluator.evaluate(evaluate_sdf)
    return score
```

#### `model_trial`、`preprocess_trial`、`train_only_func_trial`について

pyspark.ml.Pipelineを返す
trainデータを第一引数として受け取る
trialを引数として受け取る
パラメータを引数として受け取る

```
from pyspark.ml import Pipeline
from pyspark.ml.classification import LogisticRegression


def build_model_trial(
    trial: optuna.trial.Trial,
    featuresModelInputCol: str,
    labelModelInputCol: str,
    predictionCol: str
    regParam: float,
) -> Pipeline:
     lr = LogisticRegression(
        regParam=regParam,
        featuresCols=featureModelInputCol,
        labelCol=labelModelInputCol,
        predictionCol=predictionCol,
    )
    pspipeline = Pipeline(stages=[lr])
    return pspipeline
```


```
import optuna

from pyspark.ml import Pipeline
from pyspark.ml.classification import LogisticRegression


def build_model_trial(
    trial: optuna.trial.Trial,
    featuresModelInputCol: str,
    labelModelInputCol: str,
    predictionCol: str
) -> Pipeline:
    reg_param = trial.suggest_float("regParam", 0.1, 0.9)

    lr = LogisticRegression(
        regParam=reg_param,
        featuresCols=featureModelInputCol,
        labelCol=labelModelInputCol,
        predictionCol=predictionCol,
    )
    pspipeline = Pipeline(stages=[lr])
    return pspipeline
```

もし`preprocess_trial`内のパラメータも調整したい場合は、`model_trial`と同様に定義する。チューニング対象外である場合は、こちらの引数は設定する必要はない。


## アウトプットの確認

実行が完了すると、以下二つのファイルが`08_reporting`配下に生成される。

1. xxxxx
1. xxxxx

### 1. チューニングリザルト

こちらのjsonファイルに以下二つが記載されている。

データセット名：`optuna_best_result`
ファイルパス： `{experiment_path}/08_reporting/optuna/optuna_best_result.json`

1. 最良スコア
1. 最良パラメータ

中身の例
```
best_params:
    regParam: 0.18729806615604674
best_value: 0.865824015566359
```

こちらのチューニング結果をもとにパラメータファイルにパラメータの値を記入し、`use_otpuna`をオフにした上で、各種ビルドファンクションなどがその値を参照してパイプラインの形成を行うようにすれば、以降今回のチューニング結果を用いることができる。

### 2. 学習経過

こちらには`Optuna.study`が格納されており、データとして読み込んだうえでプロットしてもらえれば良い。

データセット名：`optuna_study`
ファイルパス： `08_reporting/optuna/optuna_study.pkl`
```
optuna load
```