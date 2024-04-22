# 本ページの立ち位置
このページでは，ユーザーの準備物のConfigファイル(spark.yaml)について記載する。
なおこのページで紹介するファイルは任意ファイルであり，ARISE-PIPELINEの動作に必須のものではない。
![Configの立ち位置](config_spark_position.png)
# Spark.yaml
## 概要
- Configファイルの一つでSparkの設定値を管理する。
- RAIZIN-wiki記載のデフォルト値はARISE-PIPELINEモジュール内で反映されている。デフォルト値を利用する場合は下記の「必須」項目をyamlに記載するだけでよい。
  - ARISE-PIPELINE実行時にSparkSessionが起動するため自身の分析スクリプトにSparkSessionを起動させるコードを書く必要もない。
- `additional_spark_conf`は任意の値でデフォルト値とは異なる値を利用する場合，ここにKeyとValueを記載する。
  - KeyにはSparkConfigurationにおける`property name`を, ValueにはKeyが指定する値を投入する。
| 項目 | 詳細 | 必須/任意
|-----|-----|-----|
| unit_name | ユニット名 | 必須|
| team_name | チーム名 |必須|
| executor_size |エグゼキューターサイズ。S/M/Lのいずれか|必須|
|additional_spark_conf|デフォルト設定値を上書きしたい項目。|任意|

## 記載例
以下はSynapseML0.10.1を利用するために必要な設定値をSparkConfに反映させるための記述例。
```yaml
unit_name: hoge
team_name: fuga
executor_size: S #S/M/Lのいずれか
additional_spark_conf:
    spark.jars.packages: com.microsoft.azure:synapseml_2.12.:0.10.1
```
