
# 本ページの立ち位置
このページでは，ユーザーの準備物として必要なConfigのデータカタログ(catalog.yaml)について記載する。
![Configの立ち位置](config_catalog_position.png)

# データカタログとは
## 概要
- データカタログとはARISE-PIPELINEを用いて実装する際に利用するデータ情報をまとめたものを指す。実態としては`catalog.yaml`というファイルにこの情報をまとめる。
- この考え方自体はKedroというOSSで採用されたものである。ARISE-PIPELINEはKedroをバックエンドとして採用しているためこの方法に則ってデータ情報を管理している。

## 記載方法
`catalog.yaml`にはマートパイプライン/MLパイプラインのインプットデータについて記載する。記載に必要な情報は下記となる.

| 項目 | 詳細 | 
|-----|-----|
| `type` | データの型。 |
| `filepath` | データのパス。ローカル・S3いずれも可でファイル名も含む。 |

上記以外にも指定可能な任意の引数があるがデータの型によって指定できる値が変化する。[任意項目](#任意項目)
`filepath`は名の通り読み込みたいデータのパスを記述する。`type`は種類が多いので詳細を後述する。
### type
ここではよく利用するであろう型に絞り詳細説明をする。
指定できる全量を知りたい方は[カタログに登録できるデータタイプ一覧](#カタログに登録できるデータタイプ一覧)を参照。
下記表が利用が多いと思われるデータ型をまとめた表である。

| 型名 | 詳細|用途 |
|-----|-----|-----|
| `spark.sparkDataSet` | SparkDataFrame型のデータをparquetで保存/ロード。| 前処理・特徴量生成・学習などメインのプロセスを行うとき| 
| `spark.PipelineDataSet` | SparkPypeline型のデータを保存ロードする。 | 別のパラメータで実験したときのPipelineを利用|
| `memory.MemoryDataset ` | データをメモリ上に一時的に保存するためのデータセット。|パイプライン内のノード間で中間結果を渡すときは出力を吐き出す必要がないときに利用。 |
| `pandas.CSVDataSet` | PandasDataFrame型のデータをcsv形式で保存/ロード|特徴量重要度を保持したpandas.DataFrameをcsvで保存  |
| `pandas.ParquetDataSet` | PandasDataFrame型のデータをparquet形式で保存/ロードするときに利用。 |膨大なデータを保存したいとき |
| `pandas.ExcelDataSet` | PandasDataFrame型のデータをxlsx形式で保存/ロードするときに利用。 |分析で集計した結果を報告にそのまま利用したいとき |


以下に，catalog.ymlへの記載例を示す。
```yaml
hoge_data: # hoge_dataというデータをSparkDataSetとして登録したいとき 
    type: arise_pipeline.datasets.spark.SparkDataSet # データの型。"arise_pipeline.datasets."もつけること
    filepath: data/01_raw/hoge_data.parquet # データのパス

huga_data: # huga_dataというデータをMemoryDataSetとして登録したいとき
    data: arise_pipeline.datasets.MemoryDataset
    copy_mode: "assign"

haga_data: # haga_dataというデータをCSVDataSetで登録したいとき
  type: arise_pipeline.datasets.pandas.CSVDataset
  filepath: data/02_intermediate/haga_data.csv

```
#### 任意項目について
[記載方法](#記載方法)で説明した`type`と `filepath`以外にも指定できる任意の項目が存在するが，登録するデータ型によって変動する。
具体的に何が指定できるのかを把握したい方は`cad-mau-ml-pipeline/arise_pipeline/datasets/hoge/hoge_dataset.py`にデータセット情報を記載したソースコードがある. コード内の`HogeDataSet`クラスのDocStringに指定できる引数および具体的な記述方法が記載されているのでそちらを参照してほしい。

```python
# RAIZINで必要箇所抜粋してコピペする。以下イメージ

class SparkDataSet():
    """
    ここにcatalog.yamlへの記述例がのっている。
    """

    def __init__():
        """
        Args:←ここに書いてある引数がデータカタログで指定できる。
            file_path:
            file_format:
            load_args:
            save_args:
            version:
            credentials:
        """
```
また，SparkDataSetであればサンプルノートブックの`catalog.yaml`に記載してあるのでそちらを参照してもよい。場所は`cad-mau-ml-pipeline/arise_pipeline/samples/iris/conf/logisticRegression/catalog.yml`となっている。※`iris/conf/synapseml_lightgbm/`配下でもOK

## カタログに登録できるデータタイプ一覧
下記がV1時点での登録できるデータタイプとなる。
1. json.JSONDataSet
2. matplotlib.MatplotlibWriter
3. memory.MemoryDataset
4. pandas.CSVDataSet
5. pandas.JSONDataSet
6. pandas.ParquetDataSet
7. pandas.ExcelDataSet
8. pandas.FeatherDataSet
9. pandas.GBQTableDataset
10. pandas.GenericDataset
11. pandas.HDFDataSet
12. pandas.SQLDataSet
13. pandas.XMLDataSet
14. pillow.ImageDataSet
15. plotly.JSONDataSet
16. plotly.PlotlyDataSet
17. spark.DDLDataSet
18. spark.PipelineDataSet
19. spark.sparkDataSet
20. spark.DeltaTableDataSet
21. spark.HiveDataSet
22. spark.JDBCDataSet
23. text.TextDataSet
24. yaml.TAMLDataSet