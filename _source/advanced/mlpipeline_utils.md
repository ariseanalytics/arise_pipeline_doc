# MlPipelineクラスの便利メソッド

`MlPipeline`クラスはパイプラインのビルド・実行だけでなく、いくつか便利な機能を提供している。今回はそちらを紹介する。

## SparkSessionの起動と停止
### できること
- MlPipelineクラスを介したSparkSessionの起動や停止

### 利用メソッド
- `MlPipeline.init_spark_session`

- `MlPipeline.stop_spark_session`

### 説明
`MlPipeline`クラスはSparkHookが有効になっている場合、`run`実行時に自動でSparkSessionを起動してくれ、終了時に自動で停止してくれるが、これを実行前後などで手動で起動・停止したい場合には`MlPipeline.init_spark_session`あるいは`MlPipeline.stop_spark_session`を叩けば良い。

#### 起動
```python
ml_pipeline.make(tags="train")
spark = ml_pipeline.init_spark_session()
```

#### 停止
```python
ml_pipeline.stop_spark_session()
```

### 注意事項
- こちらを利用する際には以下二つの条件を満たしておく必要がある
    1. 以下のいずれかの方法でSparkHookの設定ファイルを読み込ませること
        - `conf_path`を指定し,対象ディレクトリに「spark.yaml(.yml)」を含ませる。
        - `spark_conf`で直接yamlファイルを指定す
    1. `MlPipeline.make`によりMlPipelineをビルドさせておくこと