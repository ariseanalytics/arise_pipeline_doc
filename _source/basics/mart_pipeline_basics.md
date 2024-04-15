# マートパイプラインの基本説明

## 3つのレイヤーとワークフロー
### 全体感
マートパイプラインには以下の三つのParamsが存在し，それらを組み合わせ加工結果をマートとして保存する。なお，raw層やintermediate層などのレイヤーに関しては(LayeredThinkingのリンク)と(コンフルのレイヤールールのリンク)を参照。これら三つのレイヤーは`MartPipeline`クラスの引数として表現されており，それぞれ`MartPipelineParams`クラスを渡す必要がある。

| レイヤー名 | 立ち位置 | 役割|
| ---- | ---- |---- |
| raw2inter | 源泉データのコピー対する型付け・クレンジングを実施 | raw層のデータをを加工し，intermediate層へ設置 |
| inter2primary | 他の分析PJでも引用・加工することができるテーブルを作成 |intermediate層のデータを加工し，Primary層へ設置。 |
| primary2primary | 他の分析PJでも引用・加工することができるテーブルを作成 |Primary層のデータを加工し，Primary層へ設置。 |

## MartPipelineParamsについて
`MartPipelineParams`クラスを利用するときは下記の引数が必要となる。
- input_names: 入力のテーブル名(st)のリスト
- output_names: 出力のテーブル名(str)
- function: 出力を生成するために必要な関数。
- save: 出力を保存するか否か(boolean)。デフォルトはTrue。

下記はraw2interに渡すパラメータ`raw2inter_params`を宣言したときの例となる。
```
# raw2interで処理をする関数
def raw2iner_function(sdf):
    sdf_raw2inter_output = sdf *2 #実利用時はレイヤールールに則って該当する処理を記述。
    return sdf_raw2inter_output

# raw2interに渡すParams
raw2inter_params = MartPipelineParams(
  input_names= ['sdf_raw']
  output_name= 'sdf_raw2inter_output'
  function= raw2iner_function
  save=True)
```
inter2primaryやprimary2primaryを利用するときも上記と同じ流れで実装する。

## PathParamsについて
マートパイプラインで利用するConfigファイルやアウトプットを設置するパスまわりを管理するParamsがMartPathParamsである。
※MLパイプラインにはMlPathParamsがあるが現時点では中身は全く同じ。

- マストな引数
  - output_root_path：出力のルートを定義
  - output_subdir_order：ルートパスの下層(各実験のアウトプットパス)の規則を定義
- optionalな引数
  - project_path：プロジェクトのルートパス。ここがベースで下記のパス達はこれの配下にある。任意ではあるがデータカタログで相対パスでfilepathを指定するときは必要。
  - conf_path：yml(parametes, catalog, spark, mlflow etc)を格納しているディレクトリ名.
  - parameters_yaml_path：conf配下のパラメータファイルを個別で指定したいときに利用
  - catalog_yaml_path：conf配下のカタログファイルを個別で指定したいときに利用
  - spark_yaml_path：conf配下のsparkファイルを個別で指定したいときに利用

## インスタンス作成/実行
`MartPipeline`クラスに対して上記のParams達を引数として渡してインスタントして宣言できる。

- 必須の引数
  - path_params: MartPathParams
- optionalな引数
  - raw2inter: raw2interに属するMartPipelineParamsのリスト
  - inter2primary: inter2primaryに属するMartPipelineParamsのリスト
  - primary2primary: primary2primaryに属するMartPipelineParamsのリスト


```
#マートパイプラインのインスタンス宣言
mart_pipeline=MartPipeline(
  path_params= mart_pathparam
  raw2inter= raw2inter_params,
  inter2primary =inter2primary_params,
  primary2primary = primary2primary_params
)
```
`raw2inter`，`inter2primary`，`primary2primary`はいずれも任意であるため自身が作成したいテーブルの特性・加工内容に応じて選択する。
複数利用したい場合は，Paramsのリストを渡せばよい。
インスタンスを宣言した後，`make()`コマンドと`run()`コマンドを下記のように用いることでインスタンスの作成・実行を行うことができる。
```
mart_pipeline.make() #マートパイプラインインスタンスのビルド
mart_pipeline.run() #マートパイプライン実行
```
## 成果物の保存/バックアップに関して
`run`コマンドの実行後，`PathParams`に渡した`output_root_path`の配下に，下記の情報を加味してパスが自動で生成される。
- コマンドの実行日時
- ブランチの情報
- 実験番号(Configのyamlファイルの値から生成される)
パスの全体構造は下記の通り。
```
├── project_name
    ├── branch_name
        ├──experiment_id
            ├──01_raw
            ├──02_intermediate
            ├──03_primary
            ├──04_feature
            ├──05_model_input
            ├──06_models
            ├──07_model_output
            └──08_reporting
```
途中でエラーなく実行できれば，各種Paramsの引数`save=True`としたものに関して，ParhParamsで指定したディレクトリに結果が保存される。