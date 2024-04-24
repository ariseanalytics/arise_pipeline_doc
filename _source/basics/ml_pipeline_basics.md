# MLパイプラインの基本説明
本ページではARISE-PIPELINEモジュールにあるMLパイプラインの基本的な内容に関して説明する。
## 全体感
以下がMLパイプラインの全体構成図となっている。ユーザーはConfig，Params,Tagの三つを準備する。
![MLパイプライン構成図](mlpipeline_architect.png)
Configについては別ページで解説しているため，本ページではParams・Tag，MLパイプラインの実行方法について説明する。

## Params
### 概要
MLパイプラインには個別の役割を持つ6つのParamsが存在する。これらを組み合わせることで前処理-学習-予測-評価をワンストップで実行可能。
Params名(プロセス)とそれぞれの役割は以下表のとおり。ユーザーが実装するにあたり，実装したい処理がどのプロセスに該当するかを考えながら当てはめていく。

| プロセス名 | 役割 | 必要な実装 |
| ---- | ---- |  ---- |
| TargetUserPipeline | データの縦幅を決定 |入力データ名<br>出力データ名<br>縦幅フィルターfunction
| FeaturePipeline | 特徴量データの前処理 |入力データ名<br>出力データ名<br>ジョインキーの指定<br>特徴量の前処理function
| LabelPipeline | ラベルデータの前処理 |入力データ名<br>出力データ名<br>ジョインキーの指定<br>ラベルの前処理function
| JoinPipeline | 縦幅・特徴量・ラベルの結合 |特になし
| ModelPipeline | モデルの学習・予測 |データの分割方法<br>追加の前処理<br>モデル<br>チューニング内容
| PostPipeline | 評価・一定形式のデータ整形 |評価方法<br>後処理<br>可視化方法

また，上記に加えてパスまわりを管理するPathParamsがある。

### 注意点
各Pipelineを定義するために必要なParamsではユーザー定義functionを受け付ける仕様になっているが，functionの引数の種類として下記の3種類が存在する。

| 引数種別 | 役割 |
| ---- | ---- |  
| 必須引数 | 各functionで必ず定義する必要のある引数。<br>三つの引数のなかでは先頭に記載する必要があり，順番も守ること。
| パラメータ引数 | parameters.ymlに記述した値を参照する際に用いる引数。<br>関数側でパラメータ名と同じ引数を定義すると，その値を引数として利用可能。必須引数より後ろで定義する。
| 特殊引数 | パイプラインのプロセス側でもっている固有の値を持つ引数。functionによって使えるものが変わる。<br>必須引数より後ろで定義する。

下記の引数名は特殊引数に該当する。
| 引数名 | 型 | 詳細| 利用可能なパイプライン|
|-----|-----|-----|-----|
| `data_type` | `str`| 関数に入力するデータの種類。値はtrain/val/testのいずれか。|`TargetUserPipeline`<br>`FeaturePipeline`<br>`LabelPipeline`<br>`PostPipeline`|
| `output_path` | `str` |パイプラインの出力を保存するディレクトリパス。<br>各関数の出力先として適切なレイヤーに対応する**サブディレクトリの絶対パスを記載する。アウトプットパスのルートではない。**※こちらはユーザーが指定できる引数ではなく，内部で各レイヤーの出力先を決めるために利用している。|`raw2inter`<br>`inter2primary`<br>`primary2primary`<br>`PostPipeline`|

なお，本ページではparameters.ymlとして下記項目を定義していると仮定し，各種Paramsの説明に入る。
| 項目 |  詳細|
|-----|-----|
| `seed` | 乱数のシード|
|`train_ratio`|train/valの比率 |
|`k_pca`|PCAの次元数 ※例としてPCAを採用しておりこれ自体が必須ではない。|
|`target_data_type_value`|データの種類|
|`featuresCols`|特徴量の列名|
|`featuresVACol`|前処理(VectorAssember)をした特徴量の列名|
|`featuersModelInputCol`|モデルに入力する特徴量の列名|
|`labelCol`|ラベルの列名|
|`labelModelInputCol`|モデルに入力するラベルの列名|
|`predictionCol`|予測結果を格納する列名|
|`metrics`|評価指標(accuracy, f1など)|

### PathParams
MLパイプラインで利用するConfigファイルやアウトプットを設置するパスまわりを管理するParamsが`MlPathParams`である。
※マートパイプラインには`MartPathParams`があるが現時点では中身は全く同じ。

| 引数名 | 必須か否か|型 | 詳細|
| ---- | ---- |---- |---- |
| `output_root_path` | 必須|`Path`| 出力のルートを定義 |
| `output_subdir_order` | 必須|`Path`|ルートパスの下層(各実験のアウトプットパス)の規則を定義|
| `project_path` | 任意|`Path`|プロジェクトのルートパス。ここがベースで下記のパス達はこれの配下にある。任意ではあるがデータカタログで相対パスでfilepathを指定するときは必要。|
| `conf_path` | 任意|`Path`|yml(parametes, catalog, spark, mlflow etc)を格納しているディレクトリ名.|
| `parameters_yaml_path` | 任意|`Path`|conf配下のパラメータファイルを個別で指定したいときに利用|
| `catalog_yaml_path` | 任意|`Path`|conf配下のカタログファイルを個別で指定したいときに利用 |
| `spark_yaml_path` | 任意|`Path`|conf配下のsparkファイルを個別で指定したいときに利用 |

### TargetUserPipeline
- データの縦幅を決定(filter)するプロセス。catalog.ymlに記載されたデータをもとに，trainとtestの縦幅を決める.train/val/testで異なるデータソースを用いるときは`InputNamesTable`を用いる。
- `TargetUserPipelineParams`を利用してParamsを定義する。必要な引数は下記の通り。

| 引数名 |型 | 詳細|
| ---- | ---- |---- |
| `input_names` | `List[str]\|InputNamesTable` | データカタログで定義した入力テーブル名(str)のリスト。<br>train/val/testで異なるデータを使うときは`InputNamesTable`クラスを利用する|
| `output_name` | `str\|OutputNamesTable`|出力のテーブル名。これにデータタイプに応じた接頭辞("_train"や"_test")がつく。<br>`OutputNamesTable`を使うと独自の名前をつけられる。|
| `sdf_func` | `Callable`|`input_names`のデータを受け取り，`output_names`として出力を生成する関数。<br>使える特殊引数は`data_type`. これを利用することで共通の関数でtrain/testに対する異なる処理を実施できる。|

- ※`input_names`と`output_names`については属性値が`HogeNamesTable`となるように内部で処理しているので，いずれの型の組み合わせも利用可能。
  - 例えば`input_names`が`list[str]`で`output_names`が`OutputNamesTable`や`input_names`が`InputNamesTable`で`output_names`が`str`のような形でも可。
  

以下が`TargetUserPipelineParams`記述例。
train/val/testの分割した際に，出力は`output_table_name_{data_type}`という形で接尾辞が付いた値となる。※data_typeにはtrain/val/testのいずれかが入る。
```python
#まず縦幅フィルターを行う関数を定義
def filter_train_test(
    input_sdf: sdf,
    target_data_value_type: dict[Literal["train", "test"], str],
    data_type: Literal["train","test"]:
  ): -> sdf
    target_sdf = input_sdf.filter(
      F.col("data_type") == target_data_value_type[data_type]
    )
    return target_sdf

from arise_pipeline.ml_pipeline.main_params import TargetUserPipelineParams
# TargetUserPipelineParamsの定義。(単一のテーブルの場合)
target_user_pipeline_params = TargetUserPipelineParams(
    input_names = ["input_table_name"],
    output_name = "output_table_name"
    sdf_func = filter_func
)
```
#### InputNamesTable
`InputNamesTable`はtrain/val/testで異なるデータソースを用いるときに利用できる。引数は下記の通り。

| 引数名 |型 | 詳細|
| ---- | ---- |---- |
| `train` | `Optional[list[str]]`| trainに用いるインプットデータ名(str)のリスト |
| `val` | `Optional[list[str]]`|valに用いるインプットデータ名(str)のリスト|
| `test` | `Optional[list[str]]`|testに用いるインプットデータ名(str)のリスト|

以下が`InputNamesTable`を利用してParamsを定義するときの記述例。
前述の記述例ではデータソースが単一で分割実施前であるのに対し，こちらはtrain/val/testにすでに分割された個別のデータソースを入力している。
```python
from arise_pipeline.ml_pipeline.sub_params import InputTamesTable
# データがtrain/val/testに分割済みの場合にInputNamesTableを利用する定義方法
input_names_splitted_train_val_test = InputNamesTable(
    train=["sdf_input_train"]
    val=["sdf_input_val"]
    test=["sdf_input_test"]
)
 
target_user_pipeline_params = TargetUserPipelineParams(
    input_names = input_names_splitted_train_val_test,
    output_name = "output_table_name"
    sdf_func = filter_func
)
```

#### OutputNamesTable
- `OutputNamesTable`はアウトプットデータ名をデータタイプ(train/val/test)と対応付けるクラス。
- train/val/testとで個別のデータとして出力したいときの利用を想定している。引数は下記の通り。

| 引数名 |型 | 詳細|
| ---- | ---- |---- |
| `train` | `Optional[str]`| trainに用いるインプットデータ名(str)のリスト |
| `val` | `Optional[str]`|valに用いるインプットデータ名(str)のリスト|
| `test` | `Optional[str]`|testに用いるインプットデータ名(str)のリスト|

以下が`OutputNamesTable`を利用してParamsを定義するときの記述例。
出力としてユーザーが定義した出力値をtrain/val/testに対応づけるように吐き出す。

```python
from arise_pipeline.ml_pipeline.sub_params import OutputNamesTable
output_names_splitted_train_val_test = OutputNamesTable(
    train=["sdf_output_train"]
    val=["sdf_output_val"]
    test=["sdf_output_test"]
)
 
target_user_pipeline_params = TargetUserPipelineParams(
    input_names = ["input_table_name"],
    output_name = output_names_splitted_train_val_test,
    sdf_func = filter_func
)
```

### FeaturePipeline
- 特徴量データの前処理を行うプロセス。1つのデータソースに対して1つの前処理を対応付けて、各データソースに対してそれぞれ処理を記載する。
  - そのため，生成したい特徴量データの数だけParamsを用意する。
- Estimatorのような処理(trainデータのみにfitさせてvalデータやtestデータにはtransformさせるもの)については非該当。ModelPipelineのPreprocessで記載する。
  - ここで記載するとtestデータを含めてfitすることとなり，リークとなってしまう。
- `FeaturePipelineParams`を利用してParamsを定義する。必要な引数は下記の通り。

| 引数名 |型 | 詳細|
| ---- | ---- |---- |
| `input_names` | `List[str]\|InputNamesTable` ` | データカタログで定義した入力テーブル名(str)のリスト。<br>train/val/testで異なるデータを使うときは`InputNamesTable`クラスを利用する|
| `output_name` | `str\|OutputNamesTable`|出力のテーブル名。これにデータタイプに応じた接頭辞("_train"や"_test")がつく。<br>`OutputNamesTable`を使うと独自の名前をつけられる。|
| `sdf_func` | `Callable`|`input_names`のデータを受け取り，`output_names`として出力を生成する関数。<br>使える特殊引数は`data_type`. これを利用することで共通の関数でtrain/testに対する異なる処理を実施できる。|
| `join_keys` | `List[str]`|特徴量をジョインする際に用いる結合キーのリスト。|
| `save` | `bool`| アウトプットデータを保存するか。デフォルトは`True`|

以下の記述は2つの`FeaturePipelineParams`を一つにまとめる場合の記載例。
```python
from arise_pipeline.ml_pipeline.main_params import FeaturePipelineParams

def make_feature_pipeline_params_all():
    # 1つ目のFeaturePipelineParamsを作成する関数
    def make_feature_pipeline_params1():
        # 特徴量を作成する関数を定義する
        def feature_sdf_func1(input_sdf) -> sdf:#例としてinput_sdfの全カラムを100倍する関数
            output_sdf = input_sdf * 100
            return output_sdf

        # FeaturePipelineParamsインスタンスを作成。ここではsdf_aを入力としてfeature_sdf_func1で加工。
        feature_pspipeline_params = FeaturePipelineParams(
            input_names = ["sdf_a"],
            output_names = "sdf_a_processed",
            sdf_func=feature_sdf_func1,
            join_keys = ["hoge"],
        )

        return feature_pspipeline_params

    # 2つ目：上記と同じ流れで書く。テーブル名や処理内容は実際に扱うものに適宜変更。
    def make_feature_pipeline_params2():
        def feature_sdf_func2(input_sdf)-> sdf:#例として特定のカラムが閾値以上なら1,そうでないなら0とする関数を定義
            output_sdf = input_sdf.withColumn(
              new_column_name, 
              F.when(F.col(column_name) >= threshold, 1).otherwise(
                0
              )
            )
            return output_sdf
            
        feature_pspipeline_params = FeaturePipelineParams(
            input_names = ["sdf_b"],
            output_names = "sdf_b_processed",
            sdf_func=feature_sdf_func2,
            join_keys = ["hoge"],
        )

    # 上記メソッドを実行し，feature_pipeline_paramsのリストを作成する。
    feature_pipeline_params1 = make_feature_pipeline_params1()
    feature_pipeline_params2 = make_feature_pipeline_params2()
    
    return [feature_pipeline_params1,feature_pipeline_params2]

feature_pipeline_params_all = make_feature_pipeline_params_all()
```
### LabelPipeline
- ラベルデータの前処理を行うプロセス. catalog.ymlに記載されたデータをもとに，trainとtestのラベルを作成。
- `LabelPipelineParams`を利用してParamsを定義する。必要な引数は下記の通り。

| 引数名 |型 | 詳細|
| ---- | ---- |---- |
| `input_names` | `List[str]\|InputNamesTable` | データカタログで定義した入力テーブル名(str)のリスト。<br>train/val/testで異なるデータを使うときは`InputNamesTable`クラスを利用する|
| `output_name` | `str\|OutputNamesTable`|出力のテーブル名。これにデータタイプに応じた接頭辞("_train"や"_test")がつく。<br>`OutputNamesTable`を使うと独自の名前をつけられる。|
|`pspipeline`|`Optional[PysparkPipelineParams]`|ラベルデータに対する`PysparkPipeline`の処理内容。ラベルエンコーダーの利用を想定。<br>※`PysparkPipeline`はARISE-PIPELINEの内部でモデル学習をするために必要なPySparkのPipelineインスタンスのこと。|
| `sdf_func` | `Callable`|定義済みのラベルデータフィルター関数。。<br>使える特殊引数は`data_type`. これを利用することで共通の関数でtrain/testに対する異なる処理を実施できる。事前分割なしならtrain/testのみ。|
| `join_keys` | `List[str]`|特徴量をジョインする際に用いる結合キーのリスト。|
| `save` | `bool`| アウトプットデータを保存するか。デフォルトは`True`|

- 入力に対して，まず`sdf_func`による加工がおこなわれその出力を`pspipeline`によって加工する。ゆえにエンコーディングは`pspipeline`にて行う。
  
以下でStringIndexerを用いてラベルエンコーディングをする`LabelPipelineParams`を作成する例を記載している。
```python
from arise_pipeline.ml_pipeline.main_params import LabelPipelineParams
from arise_pipeline.ml_pipeline.sub_params import PySparlPipelineParams
from arise_pipeline.template.sdf_func impot void_sdf_func

def make_label_pipeline_params() -> LabelPipelineParams:

    def build_label_pspipeline(labelCol:str, labelModelInputCol: str) -> Pipeline:
        label_indexer = StringIndexer(InputCol=labelCol,outputCol=labelModelInputCol)
        pspipeline=Pipeline(stages=[label_indexer])
        return pspipeline

    label_pspipeline= PysparkPipelineParams(
        name="label_preprocess"
        build_func=build_label_pspipeline,
    )

    label_pipeline_params = LabelPipelineParams(
        input_names = ["sdf_labels"],
        output_names = "sdf_labels_processed",
        sdf_func=void_sdf_func,#入力をそのまま返す関数。ARISE-PIPELINEのtemplateにある。
        pspipeline=label_pspipeline
        join_keys = ["hoge"],

    )
    return label_pspipeline_params

label_pipeline_params = make_label_pipeline_params()
```

### JoinPipeline
- 縦幅，ラベル，特徴量のデータをマージするプロセス
- 定義済みのParamsの引数に渡してある`join_keys`をもとに内部で結合が行われる。そのためこのパイプラインはParamsはなく，実装する必要はない。

### ModelPipeline
- モデルの学習や予測を行うプロセス.
- JoinPipelineにてマージされたデータに対する追加の前処理やtrain/val分割なども実施.
- これまでに定義したパイプラインで処理されたtrainデータとtestデータに対して，タグに応じて挙動がかわる。
  - trainタグ：trainデータを用いてモデルの学習をする。
    - まずデータをtrain/valに分割。任意だが`train_val_split_params`を渡して分割方法を指定できる
    - (任意)train+valデータに対して前処理を実施。`preprocess_params`にて指定可。
    - (任意)trainデータに対してのみ前処理を実施。`train_only_func`にて指定可。※オーバーサンプリングなどで利用。
    - trainデータで学習し，train+valで予測する。`model_params`にて指定。
  
  - predictタグ：テストデータに対する予測を行う。
    - (任意)testデータ前処理を実施。`preprocess_params`にて指定可。
    - testデータに対して予測する。`model_params`に対応。
- Optunaによるチューニングも可能。`preprocess_params`，`train_only_func`,`model_params`のパラメータをチューニングできる。
  
- `ModelPipelineParams`を利用してParamsを定義する。必要な引数は下記の通り。


| 引数名 |型 | 詳細|
| ---- | ---- |---- |
| `model_params` | `Optional[PysparkPipelineParams]` |モデル本体の内容(任意)|
| `train_val_split_params` | `Optional[TrainValSplitParams]`|学習データのtrain/val分割方法(任意)|
|`preprocess_params`|`Optional[PysparkPipelineParams]`|train+valデータおよびテストデータに適用する前処理(任意)|
| `train_only_func` | `Optional[Callable]`|trainデータのみに適用する前処理方法(任意)|
| `use_optuna` | `bool`|Optunaを利用するか否かのフラグ。TrueでOptunaが走るがその場合は`preprocess_params`，`train_only_func`,`model_params`,`optuna_params`の指定が必要。|
| `optuna_params` | `Optional[OptunaParams]`| Optunaのパラメータ。|



以下にtrain/val分割を行い，PCA -> LogisticRegressionを行うときの記載例を記述。
```python
from pyspark.ml.feature import PCA
from pyspark.ml.classification import LogisticRegression
from arise_pipeline.ml_pipeline.main_params import ModelPipelineParams
from arise_pipeline.ml_pipeline.sub_params import TrainValSplitParams

#model_pipeline_paramsを作成するメソッド
def make_model_pipeline_params() -> ModelPipelineParams:
  # まずはデータを分割する関数を定義。
  def split_func(train_val_sdf: sdf, train_ratio: float,seed:str) -> sdf:
    train_val_sdf = train_val_sdf.withColumn(
      "is_train", #カラム名はMLパイプラインの仕様上”is_train”にする。
      F.when(F.rand(seed=seed)<=train_ratio, True).otherwise(
        False
      ), 
    )
    return train_val_sdf

    train_val_split_params = TrainValSplitParams(
      split_type= "hold-out",
      func=split_func
    )
  # 次に前処理方法
  def build_preprocess(
    k_pca: int,
    featuresVACol: str #parameters.ymlにて定義済み
    featuresModelInputCol: #parameters.ymlにて定義済み。
  ) -> Pipeline:
    pca = PCA(k=k_pca, inputCol=featuresVACol, outputCol=featuresModelInputCol)
    pspipeline = Pipeline(stages=[pca])
    return pspipeline

  preprocess_params = PysparkPipelineParams(name="model_preprocess",build_func=build_preprocess)

  # モデルの処理
  def build_model(
    featuresModelInputCol: str,
    labelModelInputCol: str,#parameters.ymlにて定義済みと仮定しているモデルに入力するラベル列名
    predictionCol: str, #parameters.ymlにて定義済みと仮定している予測列名。
  ) -> Pipeline
  lr = LogisticRegression(
    featuresModelInputCol=featuresModelInputCol
    labelModelInputCol=labelModelInputCol
    predictionCol=predictionCol
  )
  pspipeline = Pipeline(stages=[lr])
  return pspipeline

  model_params = PysparkPipelineParams(name="model_process",build_func=build_model)

  # ModelPipelineのインスタンス
  model_pipeline_params = ModelPipelineParams(
    model_params=model_params,
    train_val_split_params=train_val_split_params,
    preprocess_params=preprocess_params,
  )

  return model_pipeline_params

model_pipeline_params = make_model_pipeline_params()
```

### PostPipeline
- 評価、一定形式へのデータ整形などの後処理を行うプロセス.
- `PostPipelineParams`を利用してParamsを定義する。必要な引数は下記の通り。
  - `eval_func`: 必須。予測値のSparkDataFrame(valデータもしくはtestデータ)を受け取り，評価結果を辞書形式で返す関数。
  - `post_func`: 任意。評価以外で実施したい後処理を実施したいときに利用。`PostFuncParams`のリストを渡す

| 引数名 |型 | 詳細|
| ---- | ---- |---- |
| `eval_func` | `Callable` |予測値のSparkDataFrame(valデータもしくはtestデータ)を受け取り，評価結果を辞書形式で返す関数(必須)|
| `post_func` | `Optional[list[PostFuncParams]]`|評価以外で実施したい後処理を実施したいときに利用。`PostFuncParams`のリストを渡す(任意)|

以下はparameters.ymlのmetricsを読み取って，`MuticlasssClassificationEvaluator`を適用する際の処理を記述
```python
from pyspark.ml.evaluation import MuticlasssClassificationEvaluator
from arise_pipeline.ml_pipeline.main_params import PostPipelineParams

def make_post_pipeline_params():
  # PostPipelineParamsに渡すeval_funcを定義
  def eval_func(
    evaluate_sdf: sdf,#予測値のSparkDataFrame
    labelModelInputCol: str, # parametersで定義しているやつ。predictionCol/metricsも同様。
    predictionCol: str,
    metrics: dict[str, str]
  ) -> dect[str, float]:
    eval_results={}
    for metric in metrics:
        evaluator = MuticlasssClassificationEvaluator(
          labelCol=labelModelInputCol,
          predictionCol=predictionCol,
          meticName=metric,
        )
    eval_results[metric]=evaluator.evaluate(evaluate_sdf)
    return eval_results
  
  post_pipeline_params = PostPipelineParams(eval_func=eval_func)
  return post_pipeline_params

post_pipeline_params = make_post_pipeline_params()
```
## Tag
### 概要
- 「train」「predict」「evaluate」の3つがあり，指定したタグに応じて利用データと走るプロセスが変わる。
  - trainタグはtrain+valに対して、他2つはtestデータに対して処理が行われる。
- そのため後述する実行コマンドを叩く際に与えるタグを変えるだけで学習/予測/評価の切り替えが可能でそれぞれ個別にコードを書く必要がない。

- タグとデータの対応関係は下記のとおり。それぞれのタグに応じて扱われるデータやラベルが必要か否か変動する。
  
|      |trainデータ | valデータ| testデータ |
| ---- | ---- |  ---- | ---- | 
|利用タグ |train | train| predict/evaluate |
|用途 | モデルの学習| early stopping<br>ハイパラチューニング<br>スタッキング時のoof値算出 |本番運用時の予測値算出<br>精度算出
|ラベルの要・不要 |必要 |必要 |なくても可|

- タグとParamsの関係性は以下の通り。なお，パイプライン実行に指定するタグは単一/複数いずれも可能。

| Params名 | 役割 | train | predict| evaluate |
| ---- | ---- |  ---- | ---- |  ---- |
| `TargetUserPipelineParams` | データの縦幅を決定 |○|○| -|
| `FeaturePipelineParams` | 特徴量データの前処理 |○|○| -|
| `LabelPipelineParams` | ラベルデータの前処理 |○|○| -|
| `JoinPipelineParams`| 縦幅・特徴量・ラベルの結合 |○|○| -|
| `ModelPipelineParams` | モデルの学習・予測 |○|○| -|
| `PostPipelineParams` | 評価・一定形式のデータ整形 |○|○| ○|

## インスタンス作成/実行
`MlPipeline`クラスに対して上記のParams達を引数として渡してインスタンスとして宣言できる。

|引数名 |必須か否か|詳細|
| ---- | ---- | ---- | 
|`path_params` |必須|`PathParams`を受け取る。 |
|`target_user_pipeline_params`|任意|`TargetUserPipelineParams`を受け取る。|
|`feature_pipeline_params_all`|任意|`FeaturePipelineParams`のリストを受け取る。|
|`label_pipeline_params`|任意|`LabelPipelineParams`を受け取る|
|`model_pipeline_params`|任意|`ModelPipeline`を受け取る。|
|`post_piepline_params`|任意|`PostPipelineParams`を受け取る。|

```python
# 記載例
ml_pipeline = MlPipeline(
    path_params = path_params,
    target_user_pipeline_params=target_user_pipeline_params
    feature_pipeline_params_all=feature_pipeline_params_all
    label_pipeline_params=label_pipeline_params
    model_pipeline_params=model_pipeline_params
    post_piepline_params=post_piepline_params
)
```

インスタンスを宣言した後，`make()`コマンドと`run()`コマンドを下記のように用いることでインスタンスの作成・実行を行うことができる。
`make()`コマンドでは引数`tags`でタグのリストを渡す。使い分けは下記の通り。
|tag |使い方|
| ---- | ---- | 
|train |trainデータに対して全パイプラインを実施 |
|predict |testデータに対してModelPipelineまでを実施|
|evaluate |testデータに対してPostPipelineを実施 |
```python
ml_pipeline.make(tags=["train","predict","evaluate"]) #MLパイプラインインスタンスのビルド
ml_pipeline.run() #MLパイプライン実行
```
`run()`コマンドを実行しエラーがなければ，`PathParams`で指定したディレクトリに結果が保存される。