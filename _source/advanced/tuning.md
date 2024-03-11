# Optunaを用いたパラメータチューニング

## はじめに

MLPipelineには、標準でOptunaによるパラメータチューニング機能を搭載している。ここではそちらの使い方を紹介する。

### できること
- モデルパイプラインに含まれる各種パラメータのチューニング
    - `ModelPipelineParams.main_model`
    - `ModelPipelineParams.preprocess_model`
    - `ModelPipelineParams.train_only_func`
- チューニング経過・結果の保存・読み込み
    - チューニング経過（`optuna.study.Study`）
    - 最良パラメータ（`optuna.study.Study.best_params`）
    - 最良スコア（`optuna.study.Study.best_value`）

### できないこと

- モデルパイプライン以外に含まれる各種パラメータのチューニング

※ただし、自前でOptunaのチューニングコードを書いて、`MlPipeline`自体を包む形でチューニングを行うことは可能

## 利用方法

### おおまかな流れ

まず、全体の流れは次の通りである。今回は2のみを説明する。

1. `ModelPipelineParams`以外の各種paramsの準備や設定ファイルの準備を行う
1. `ModelPipelineParams`中でチューニングコードや条件を定義する
1. `MlPipeline`クラスのビルド・実行

### `ModelPipelineParams`の設定

基本的な`ModelPipelineParams`の使い方に加えて、チューニングを行う際に守るべき条件は以下の通りである。

- `train_val_split_params`は必ず定義しておく
    - Optunaのチューニングには目的関数となるスコアの算出に用いるvalidationデータが必須なので、`train_val_split_params`を定義しておく
- `model_params`、`preprocess_params`のうちチューニング対象となるものやチューニング対象でなくともパイプラインに組み込むものについては、必ず定義しておく
    - 基本的には、チューニング前のパラメータで`MlPipeline`を正しく動かせるものを定義しておけばよい
    - 仕組みとしては、後述の`OptunaParams`の`model_trial`や`preprocess_trial`で定義されているものについては、チューニングが走り、そうでないものに関しては、こちらで定義された内容のビルドファンクションなどが走る
- `use_optuna = True`とする
    - こちらを`True`にしておくことでチューニングが実施され、`False`の場合はチューニングが実施されないため、必ず`True`にしておくこと。
- `optuna_params`に用意した`OptunaParams`を渡す
    - 具体的なチューニング内容についてはこちらで定義する（後述）


例えば、以下のような条件のチューニングを行う場合の、パラメータの定義は以下の疑似コードのようになる。

- 条件
    - `model_params`と`preprocess_params`はあり、`train_only_func`はなし
    - `model_params`のみがチューニング対象で、`preprocess_params`はチューニング対象でない
- 疑似コード

    ```python
    from arise_pipeline.ml_pipeline.sub_params import OptunaParams
    from arise_pipeline.ml_pipeline.main_params import ModelPipelineParams


    # Optunaのパラメータ
    optuna_params = OptunaParams(
        optimize_params=optimize_params,
        create_study_params=create_study_params,
        model_trial=build_model_trial, # preprocessはチューニング対象
        preprocess_trial=None, # preprocessはチューニング対象でないのでNone
        train_only_func_trial=None, # そもそもパイプラインに含まれていないのでNone
        score_func=score_func,
    )

    # モデルパイプラインのパラメータ
    model_pipeline_params = ModelPipelineParams(
        model_params=model_params, # 必須
        preprocess_params=preprocess_params, # チューニング対象ではないが、パイプラインの構成には含まれるので定義
        train_only_func=None, # パイプラインの構成には含まれないのでNone
        use_optuna=True, # チューニングを実施するのでTrue
        optuna_params=optuna_params, # チューニングの条件を渡しておく
    )
    ```

### `OptunaParams`の設定の全体感

基本的にチューニングの条件については、`OptunaParams`に渡す引数で全て完結する。ざっくりと各引数について説明する。

| `OptunaParams`の引数 | 説明 |
| ---- | ---- |
| `optimize_params` | `study.optimize`に引き渡す引数を辞書（`dict[str, Any]`）で定義したもの |
| `create_study_params` | `create_study`に引き渡す引数を辞書（`dict[str, Any]`）で定義したもの |
| `score_func` | バリデーションデータに対する予測結果のデータフレームを受け取り、最小化あるいは最大化したいスコアを返す関数を定義したもの。Optunaにおけるobjective関数と思っていただければよい。 |
| `model_trial` | `ModelPipelineParams`の`main_params`に渡す`build_func`をチューニングする場合に用いる。Optunaの`trial`を受け取れる`build_func`を定義することで動的にパラメータを生成させる。 |
| `preprocess_trial` | `model_trial`と同様で、`ModelPipelineParams`の`preprocess_params`のチューニング用引数。 |
| `train_only_func` | `model_trial`と同様で、`ModelPipelineParams`の`train_only_func`のチューニング用引数。 |

#### `optimize_params`

`study.optimize`に引き渡す引数を辞書（`dict[str, Any]`）で定義すればよく、例えば試行回数が20回という条件のみ指定したい場合は以下のようなものを定義する。

```python
optimize_params = {"n_trials": 20}
```

#### `create_study_params`

 `create_study`に引き渡す引数を辞書（`dict[str, Any]`）で定義すればよく、例えば目的関数を大きくしたい＆studyの名前を"iris_sample_tuning"にしたい場合は以下のようなものを定義する。

```python
create_study_params = {
    "direction": "maximize",
    "study_name": "iris_sample_tuning",
}
```

#### `score_func`

最小化あるいは最大化したいスコアを吐き出す関数を定義すればよい。先頭引数（`evaluate_sdf`）にバリデーションデータに対する予測結果が格納されているので、こちらをもとに何らかのスコア（`float`）を算出するコードを書けばよく、例えばf1スコアを最大化させたい場合は以下のようなコードとなる。


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

なおパラメータファイル（yaml）には以下のようなパラメータが定義されている前提とする。

```yaml
labelModelInputCol: labelInput
predictionCol: prediction
metricName: f1
```

#### `model_trial`、`preprocess_trial`、`train_only_func_trial`について

「`model_trial`」「`preprocess_trial`」「`train_only_func_trial`」の3つについては、それぞれ以下の対応関係にある関数をチューニングするための引数となっている。

| `OptunaParams`の引数 | `ModelPipelineParams`のチューニング対象 |
| ---- | ---- |
| `model_trial` | `model_params`の`build_func` |
| `preprocess_trial` | `preprocess_params`の`build_func` |
| `train_only_func_trial` | `train_only_func` |

いずれも、チューニング対象の関数に＋αで「`trial`」という引数を受け取り、これを用いて内部で`Optuna`ライクにパラメータを生成することでチューニングコードを記載する。

また、ここではチューニングを行いたい関数のみを定義すればよく、チューニング対象でない場合は定義する必要はない。

例えば、`model_params`のみをチューニング対象とする場合は以下のような関数を定義し、 `model_trial`に渡してあげればよい

- `model_trial`に渡してあげる関数のコード例

```python
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

- （参考）上記チューニング対象の元の関数のイメージ

```python
from pyspark.ml import Pipeline
from pyspark.ml.classification import LogisticRegression


def build_model(
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


もし`preprocess_trial`など他の関数もチューニング対象にしたい場合は、`model_trial`と同様に定義する。

## アウトプットの確認

前述までの内容をもとに定義した`MlPipeline`を実行するとチューニングが実施され、最終的に以下二つの成果物が`{experiment_path}/08_reporting/optuna`配下に生成される。

1. チューニングリザルト（`optuna_best_result.json`）
1. チューニング経過（`optuna_study.pkl`）

これらについて説明する。

### 1. チューニングリザルト

#### データセットについて
- データカタログ上の名前：`optuna_best_result`
- ファイルパス： `{experiment_path}/08_reporting/optuna/optuna_best_result.json`

#### 説明

こちらには以下二つの内容が含まれており、`optuna.study.Study`の`best_params`と`best_value`をそれぞれ辞書に代入したものをjsonファイルとして書き出したものである。

1. 最良スコア
    - key: `best_params`
    - value: `optuna.study.Study.best_params`
1. 最良パラメータ
    - key: `best_value`
    - value: `optuna.study.Study.best_value`

以下は中身の例である。

```json
best_params:
    regParam: 0.18729806615604674
best_value: 0.865824015566359
```

こちらのチューニング結果をもとにパラメータファイルにパラメータの値を記入し、`use_otpuna`をオフにした上で、各種ビルドファンクションなどがその値を参照してパイプラインの形成を行うようにすれば、以降今回のチューニング結果を用いることができる。

### 2. チューニング経過

#### データセットについて
- データカタログ上の名前：`optuna_study`
- ファイルパス： `08_reporting/optuna/optuna_study.pkl`

#### 説明
こちらには`optuna.study.Study`そのものが格納されているため、こちらのデータをロードしていただければ、チューニング経過に対する深堀が可能である。そちらについては`Optuna`の公式ドキュメントなどを参考にしていただきたい。
