# MlPipeline/MartPipelineクラスの便利機能

`MlPipeline`クラスおよび`MartPipeline`クラスはパイプラインのビルド・実行だけでなく、いくつか便利な機能を提供している。今回はそちらを紹介する。基本的にはいずれのメソッドも`make`によるビルドまでが済んでいることが利用の前提となるので注意。
なお，本ページで紹介するメソッドは`MartPipeline`クラスのアトリビュートとして同様の呼び出し方で同様の結果が得られるようになっている。そのため"利用メソッド"セクションで記述しているものは`MlPipeline`クラス・`MartPipeline`クラスどちらでも利用可能。

## データカタログの確認
### できること
- データカタログ（インプットやアウトプットデータのパスやデータ形式）の確認

### 利用メソッド
- `data_catalog`
- `inputs`
- `input_names`
- `outputs`
- `output_names`


### 説明
パイプラインのデータカタログ情報（インプットやアウトプットがどのようなパスにどのようなファイル形式で登録されているか）を確認するためにいくつかメソッドが用意されている。成果物の確認やカタログファイルを編集する際の参考になると思われる。また、後述の、[データの読み込み](#データの読み込み)の際に、データセット名を確認するためにも役立つ。

#### 全データの確認
中間生成物含め、インプット・アウトプット全てのデータカタログ情報を取得するには、 `data_catalog`を利用すればよい。

- サンプルコード
```python
ml_pipeline.data_catalog()
```

データセット名がkey、データの詳細がvalueとなる辞書(`dict[str, Any]`)が返ってくる。

- 実行結果
```python
{
    "iris_targets_processed_train": {
        "type": "arise_pipeline.datasets.spark.SparkDataset",
        "filepath": "s3a://hoge/fuga/04_feature/iris_targets_processed_train",
        "save_args": {"sep": ",", "header": True, "mode": "overwrite"},
        "load_args": {"header": True, "inferSchema": True},
        "file_format": "parquet",
    },
    "iris_targets_processed_test": {
        "type": "arise_pipeline.datasets.spark.SparkDataset",
        "filepath": "s3a://hoge/fuga/04_feature/iris_targets_processed_test",
        "save_args": {"sep": ",", "header": True, "mode": "overwrite"},
        "load_args": {"header": True, "inferSchema": True},
        "file_format": "parquet",
    },
    ...
}
```


#### 特定のデータカタログの確認
特定のデータのみ確認する場合は `data_catalog`の引数`names`にデータセット名のリストを渡してあげればよい。(複数可)

- サンプルコード
```python
ml_pipeline.data_catalog(names=["iris_targets_processed_test"])
```

- 実行結果
```python
{
    "iris_targets_processed_test": {
        "type": "arise_pipeline.datasets.spark.SparkDataset",
        "filepath": "s3a://hoge/fuga/04_feature/iris_targets_processed_test",
        "save_args": {"sep": ",", "header": True, "mode": "overwrite"},
        "load_args": {"header": True, "inferSchema": True},
        "file_format": "parquet",
    }
}
```


#### インプットの確認
インプットデータのみを確認したい場合は以下コマンドを叩けばよい。実行結果は`data_catalog`と同様のフォーマットであるため割愛。

- サンプルコード
```python
ml_pipeline.inputs()
```

また、データカタログではなく名前だけ確認する場合は以下コマンドを用いると、名前だけ取り出せる。

- サンプルコード
```python
ml_pipeline.input_names()
```

インプット名の集合(`set[str]`)が返ってくる。

- 実行結果
```python
{"iris_features", "iris_labels", "iris_targets"}
```

#### アウトプットの確認
アウトプットデータ（中間生成物含む）のみを確認したい場合はインプットの時と同じ要領で以下コマンドを叩けばよい。

- サンプルコード
```python
ml_pipeline.outputs()
```

名前のみの取り出しの場合は以下コマンドを叩く。

- サンプルコード
```python
ml_pipeline.output_names()
```

## データの読み込み
### できること
-データカタログに登録されているデータの読み込み

### 利用メソッド
- `load`

### 説明
インプットデータやアウトプットデータについて、データカタログ上のデータセット名を指定することで、読み込むことができる。これによって、成果物のデータフレームを直接操作して中身を確認したりすることができる。

- サンプルコード
```python
test_scores = ml_pipeline.load(name="test_scores")
test_scores
```

- 実行結果
```python
[03/05/24 07:51:31] INFO Loading data from 'test scores' (JSONDataSet)...
{'accuracy': 0.8, 'f1': 0.8}
```

データセット名が不明な場合は、[データカタログの確認](#データカタログの確認)で紹介した各種メソッドを用いて確認すると良い。

## パイプライン・ノードの確認
### できること
- ビルドされたパイプライン・ノードの確認

### 利用メソッド
- `show`
- `nodes`
- `node_names`

### 説明
ビルドされたパイプラインが実際にどのようなノードの集合で形成されているかを確認するためのメソッドがいくつかある。

#### パイプラインの全体感の確認
パイプライン全体を確認するためには以下コマンドを実行すればよい。

- サンプルコード
```python
print(ml_pipeline.show())
```

なお、`print`で表示した方が改行の関係で見やすいので、そちらを利用することをすすめる。将来的にはもう少しノードの関係を見やすい画像形式での表示などを考えている。現状は以下のようにパイプライン全体の「Inputs」「Outputs」並びに各ノード名が縦に並ぶ形となっている。

- 実行結果
```python
#### Pipeline execution order ####
Inputs: iris_features, iris_labels, iris_targets, parameters, ....
feature_sdf_func([iris_features,iris_labels,params:featureCols,...]) -> [iris_features_processed_test]
...
Outputs: test_scores, train_scores, val_scores
##################################
```

#### ノードの詳細の確認
各ノードの詳細を確認するためには以下コマンドを使うと良い。

- サンプルコード
```python
ml_pipeline.nodes()
```

以下のようにノード名がkey、ノードの詳細がvalueとなる辞書(`dict[str, Any]`)が返ってくる。`'feature_sdf_func([iris_features,iris_labels,params:featureCols,params:featuresCols,params:featuresVACol]) -> [iris_features_processed_test]'`がノード名であることに注意。

- 実行結果
```python
{
    'feature_sdf_func([iris_features,iris_labels,params:featureCols,params:featuresCols,params:featuresVACol]) -> [iris_features_processed_test]': {
        'func_name' : 'feature_sdf_func',
        'inputs': ['iris_features', 'iris_labels', 'params:featureCols', 'params:featuresVACol'],
        'outputs': ['iris_features_processsed_test'],
        'tags': ['predict'],
        'procs': ['pre', 'feature'],
    },
    ...
}
```


#### ノード名の確認
ノード名だけを確認したい場合は、以下コマンドを使うと良い。

- サンプルコード
```python
ml_pipeline.node_names()
```

ノード名の集合(`set[str]`)が返ってくる。

- 実行結果
```python
{
    'feature_sdf_func([iris_features,iris_labels,params:featureCols,params:featuresCols,params:featuresVACol]) -> [iris_features_processed_test]',
    ...
}
```

## SparkSessionの起動と停止
### できること
- SparkSessionの起動や停止

### 利用メソッド
- `init_spark_session`
- `stop_spark_session`

### 説明
`MlPipeline`クラスはSparkHookが有効になっている場合、`run`実行時に自動でSparkSessionを起動してくれ、終了時に自動で停止してくれるが、これを実行前後などで手動で起動・停止したい場合には`init_spark_session`あるいは`stop_spark_session`を叩けば良い。

#### 起動
事前にビルドしておき、以下コマンドを叩けば、SparkSessionが起動する。

- サンプルコード
```python
spark = ml_pipeline.init_spark_session()
```

- 実行結果
```python
...
[03/05/24 07:51:31] INFO SparkSessionを起動しました.
...
```

#### 停止
停止するには以下のように実行。

- サンプルコード
```python
ml_pipeline.stop_spark_session()
```

- 実行結果
```python
[03/05/24 07:52:30] INFO SparkSessionを停止しました.
```

### 注意事項
- こちらを利用する際には、以下のいずれかの方法でSparkHookの設定ファイルを読み込ませておくこと
    - `conf_path`を指定し,対象ディレクトリに「spark.yaml(.yml)」を含ませる
    - `spark_conf`で直接yamlファイルを指定する