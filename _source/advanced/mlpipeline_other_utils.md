# MlPipelineクラスの便利機能

`MlPipeline`クラスはパイプラインのビルド・実行だけでなく、いくつか便利な機能を提供している。今回はそちらを紹介する。基本的にはいずれのメソッドも`MlPipeline.make`によるビルドまでが済んでいることが利用の前提となるので注意。

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
パイプラインのデータカタログ情報（インプットやアウトプットがどのようなパスにどのようなファイル形式で登録されているか）を確認するためにいくつかメソッドが用意されている。成果物の確認やカタログファイルを編集する際の参考になると思われる。また、後述の、[データの読み込み](#データの読み込み)の際に、データセット名を確認するためにも役立つ。

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

## データの読み込み
### できること
-データカタログに登録されているデータの読み込み

### 利用メソッド
- `MlPipeline.load`

### 説明
インプットデータやアウトプットデータについて、データカタログ上のデータセット名を指定することで、読み込むことができる。これによって、成果物のデータフレームを直接操作して中身を確認したりすることができる。

```python
val_preds_sdf = ml_pipeline.load("val_preds")
```

データセット名が不明な場合は、[データカタログの確認](#データカタログの確認)で紹介した各種メソッドを用いて確認すると良い。

```python
ml_pipeline.outputs()
```

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


## パイプライン・ノードの確認
### できること
- ビルドされたパイプライン・ノードの確認

### 利用メソッド
- `MlPipeline.show`
- `MlPipeline.nodes`
- `MlPipeline.node_names`

### 説明
ビルドされたパイプラインが実際にどのようなノードの集合で形成されているかを確認するためのメソッドがいくつかある。

#### パイプラインの全体感の確認
パイプライン全体を確認するためには以下コマンドを実行すればよい。なお、`print`で表示した方が改行の関係で見やすいので、そちらを利用することをすすめる。将来的にはもう少しノードの関係を見やすい画像形式での表示などを考えている。

```python
ml_pipeline.show()
```

#### ノードの詳細の確認
各ノードの詳細を確認するためには以下コマンドを使うと良い。

```python
ml_pipeline.nodes()
```

#### ノード名の確認
ノード名だけを確認したい場合は、以下コマンドを使うと良い。

```python
ml_pipeline.node_names()
```