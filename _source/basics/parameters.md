
# 本ページの立ち位置
このページでは，ユーザーの準備物として必要なConfigのパラメータ(parameters.yaml)について記載する。
![Configの立ち位置](config_parameters_position.png)
# パラメータ
## 概要
ARISE-PIPELINEでは学習等で用いるパラメータをは`parameters.yaml`というファイルに記載する。ファイルで一括管理することで，スクリプトのボリュームが減り保守性が向上。設定値を確認する際にこのファイルを閲覧するだけでよく，検索性も高い。

## 注意事項
- Paramsに渡す関数の引数にて，`parameters.yaml`に記載した値と同名の引数を定義するとyamlファイルの値を参照できる。
- 一部特殊引数として指定されたものがあり，その名前はパラメータ名に指定できない。

## parameters.yamlの記載方法
ここは一般的なyamlファイルに値を書くように記載する。特殊引数を除けばキーの値は自由。

```
train_ratio: 0.8 # train/valの比
labelCol: target # 目的変数の列名
featuresCols: # 特徴量の列名
  - col_name_1
  - col_name_2
  - col_name_3  
```

以下に指定したSparkDataFrameからfeaturesColsに記載した列名を抽出する関数を利用する場合の例を記述する。
```
# your_notebook.ipynb に記載
def sample_function(input_sdf, featuresCols): # featuresColsはparameters.ymlに記載。
    selected_sdf = input_sdf.select(featuresCols)
    return selected_sdf

raw2inter_params = MartPipelineParams(
  input_names= 'input_table'
  output_names= 'output_table'
  function= sample_function
  save=True)

```

## 特殊引数について
前提として各Paramsに渡すユーザー定義functionには以下の三種類が存在する。
- 必須引数
  - 各functionで必ず定義する必要のある引数。
  - 三つの引数のなかでは先頭に記載する必要があり，順番も守ること。
- パラメータ引数
  - `parameters.yaml`に記述した値を参照する際に用いる引数。
  - 関数側でパラメータ名と同じ引数を定義すると，その値を引数として利用可能。必須引数より後ろで定義する。
- 特殊引数
  - パイプラインのプロセス側でもっている固有の値を持つ引数。functionによって使えるものが変わる。
  - 必須引数より後ろで定義する。

下記の引数名は特殊引数に該当する。
| 引数名 | 型 | 詳細| 利用可能なパイプライン|
|-----|-----|-----|-----|
| `data_type` | str| 関数に入力するデータの種類。train/val/testのいずれか。|TargetUserPipeline<br>FeaturePipeline<br>LabelPipeline<br>PostPipeline|
| `output_path` | str |パイプラインの出力を保存するディレクトリパス。各関数の出力先として適切なレイヤーに対応する**サブディレクトリの絶対パスを記載する。**※アウトプットパスのルートではない|raw2inter<br>inter2primary<br>primary2primary<br>PostPipeline|




