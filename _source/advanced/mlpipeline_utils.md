# MlPipelineクラスの便利メソッド

`MlPipeline`クラスはパイプラインのビルド・実行だけでなく、いくつか便利な機能を提供している。今回はそちらを紹介する。

## データカタログの確認
### できること
- データカタログ（インプットやアウトプットデータのパスやデータ形式）の確認

### 利用メソッド
- `MlPipeline.data_catalog`
- `MlPipeline.inputs`
- `MlPipeline.input_names`
- `MlPipeline.outputs`
- `MlPipeline.output_names`


### 説明
パイプラインのデータカタログ情報（インプットやアウトプットがどのようなパスにどのようなファイル形式で登録されているか）を確認するためにいくつかメソッドが用意されている。成果物の確認やカタログファイルを編集する際の参考になると思われる。

#### 全データの確認
中間生成物含め、インプット・アウトプット全てのデータカタログ情報を取得するには、 `MlPipeline.data_catalog`を利用すればよい。
```python
ml_pipeline.data_catalog()
```

#### インプットの確認
インプットデータのみを確認したい場合は以下コマンドを叩けばよい。
```python
ml_pipeline.inputs()
```

また、データカタログではなく名前だけ確認する場合は以下コマンドを用いると、名前だけ取り出せる。
```python
ml_pipeline.input_names()
```

#### アウトプットの確認
アウトプットデータ（中間生成物含む）のみを確認したい場合はインプットの時と同じ要領で以下コマンドを叩けばよい。
```python
ml_pipeline.outputs()
```

名前のみの取り出しの場合は以下コマンドを叩く。
```python
ml_pipeline.output_names()
```

### 注意事項
- こちらを利用する際には `MlPipeline.make`によりMlPipelineをビルドさせておく必要がある


## SparkSessionの起動と停止
### できること
- MlPipelineクラスを介したSparkSessionの起動や停止

### 利用メソッド
- `MlPipeline.init_spark_session`

- `MlPipeline.stop_spark_session`

### 説明
`MlPipeline`クラスはSparkHookが有効になっている場合、`run`実行時に自動でSparkSessionを起動してくれ、終了時に自動で停止してくれるが、これを実行前後などで手動で起動・停止したい場合には`MlPipeline.init_spark_session`あるいは`MlPipeline.stop_spark_session`を叩けば良い。

#### 起動
事前にビルドしておき、以下コマンドを叩けば、SparkSessionが起動する。
```python
ml_pipeline.make(tags="train")
spark = ml_pipeline.init_spark_session()
```

#### 停止
停止するには以下のように実行。
```python
ml_pipeline.stop_spark_session()
```

### 注意事項
- こちらを利用する際には以下二つの条件を満たしておく必要がある
    1. 以下のいずれかの方法でSparkHookの設定ファイルを読み込ませること
        - `conf_path`を指定し,対象ディレクトリに「spark.yaml(.yml)」を含ませる。
        - `spark_conf`で直接yamlファイルを指定す
    1. `MlPipeline.make`によりMlPipelineをビルドさせておくこと
