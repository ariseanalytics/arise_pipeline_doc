# 本ページの立ち位置
このページでは，ユーザーの準備物のConfigファイル(mlflow.yaml )について記載する。
なおこのページで紹介するファイルは任意ファイルであり，ARISE-PIPELINEの動作に必須のものではない。
![Configの立ち位置](config_spark_position.png)

# MLFlow.yaml
## 概要
- Configファイルの一つでMLFlowをARISE-PIPELINEで利用する場合に使用。
- 他のyamlファイルのようにMLFlowに関連する設定値を記載する。より詳細な理解を深めたい方はMLFlowの公式ドキュメントを読むことを推奨。

| 引数名 | 必須か否か | 詳細|
| ---- | ---- |---- |
| `tracking_uri` | 必須 |MLFlowサーバーのアドレス。このURIを設定すると実験の結果を追跡しMLFlowに記録する場所を指定できる。アドレスはチームごとに割り振られているので該当するアドレスを記載する。
| `experiment_args` | 必須 | `mlflow.create_experiment`に渡される引数。`name`,`artifact_location`,`tags`の三つを渡すことができるが必須なのは`name`のみ。 |
| `run_args` | 任意 | `mlflow.start_run`に渡される引数. 基本的に`run_name`と`tags`を指定してあげればよい。|
| `log_parameters` | 任意 |MLflowにおける`parameters`としてログを行う対象を記載.それぞれの項目についてはboolで指定する。ログの対象としてはV1.0.0では下記のとおり<br>・`parameters`: parameters.yamlに記載してあるパラメータ。全選択もしくは部分的な選択どちらでもOK。<br>・`experiment_path`: ARISE-PIPELINEが発行するexperiment_path|
| `log_tags` | 任意 | MLflowにおける`tags`としてログを行う対象を記載. ログの対象としてはV1.0.0では下記のとおり<br>・`make_tags`: `make`コマンドに用いたtrain/predict/evaluateのタグ|
| `log_metrics` | 任意 | `mlflow.log_metric`でログを行う対象を記載. ログの対象としてはV1.0.0では下記のとおり<br>・`scores`: `PostPipeline`における`eval_func`の吐き出すスコア.train/val/testそれぞれに対して個別指定も可能。<br>・`nodes_status`: ノードの成功数、エラー数など|

以下は記述例
```yaml
tracking_uri: http://192.168.1.0 #自チームの利用が承諾されているURIを入力する

experiment_args:
  name: test_experiment #実験の名前。
  artifact_location: s3a://yourbucketname/path/to/artifacts #任意

run_args:
  run_id: #実行時に実験付与されるUUID
  run_name: #実験の名前。run_id が指定されていない場合にのみ使用
  tags: #実行時にタグして設定するキー(文字列)と値の辞書。
    tag1: hoge 
    tag2: fuga
  nested: False

log_parameters:
  parameters: #parametersファイル記載のものを書く。"parameters:True"と記載すると全てのパラメータについてロギング。
    seed: True
    train_ratio: True
  experiment_path: True # ARISE-PIPELINEが発行するexperiment_path

log_tags:
    make_tags: True
log_metrics:
  scores: #PostPipelineのeval_funcで出職するスコア。"scores: True"と書けば全てのロギングする。
    train: True
    val: False
    test: True
  nodes_status: True
```

実行後，`tracking_uri`に指定したURLにブラウザでアクセスすると上記の設定値が反映されたmlflowのWEBUIが表示される。
※RAIZINでサンプル画像挿入↓