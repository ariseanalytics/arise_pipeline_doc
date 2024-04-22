# MlPipeline/MartPipelineクラスのmakeコマンド応用編
本ページでは2つのPipelineクラスが備えている`make`コマンドの応用的な使い方を解説する。
Pipelineインスタンスの作成に必要なParamsが既に定義できている前提で進める。

## 前提
### ノードとプロセスの関係について
本編を理解するために把握しておくべき概念であるノードとプロセス(パイプライン)について述べる。
ノードとは，パイプラインを構成する最小単位の処理のことを指し，下記三点で構成される。
| 引数名 | 概要|
| :------: | :------- |
|inputs  |関数の入力として使用する変数名。   |
| func  | inputsに対して処理を行う関数。 |
|outputs | funcの出力として使用される変数名 |

このノードを連結することでパイプラインができあがり，パイプラインを連結することでマート作成や学習・予測といった一連の処理を実現することができる。
なお，この考え方はARISE-PIPELINEのバックエンドとなっているOSSのKedroの考え方である。
より理解を深めたい方はKedroの公式ドキュメントを参照を推奨。

### 前提
本ページで解説するにあたって，下記のデータを扱っているとする。
| 引数名 | 概要|
| :------: | :------- |
|inputs  |関数の入力として使用する変数名。   |
| func  | inputsに対して処理を行う関数。 |
|outputs | funcの出力として使用される変数名 |


## makeコマンドの引数一覧
下記の表に`make`コマンドに付帯する引数の一覧を記載する。procsとつく引数については指定できる値が異なるので注意されたし。
この引数を駆使することで実装時における特定のノードのみを実行したい，といったことも実現できる。
以下の表はMLパイプラインクラスにおける引数表である。
| 引数名 | 型|概要|
| :------: | :------- | :------- |
| `procs` | `Iterable[Literal["pre","target","feature","label","join","model","post"]]`|特定のプロセス「のみ」を実行したいときに使う。 |
|`to_procs`  |`Iterable[Literal["pre","target","feature","label","join","model","post"]]`| 特定のプロセス「まで」を実行したい場合に用いる。  |
|`from_procs` |`Iterable[Literal["pre","target","feature","label","join","model","post"]]`|特定のプロセス「以降」を実行したい場合に用いる。   |
|`to_nodes`  |`Iterable(str)`|特定のノード「まで」を実行したい場合に用いる。   |
|`from_nodes` |`Iterable(str)`|特定のノード「以降」を実行したい場合に用いる   |
|`node_names` | `Iterable(str)`|特定のノードを実行したい場合に用いる。  |
|`from_input` | `Iterable(str)`|特定のインプット以降のノードを実行したい場合に用いる。 |
|`to_outputs` | `Iterable(str)`|特定のアウトプットまでのノードを実行したい場合に用いる  |

以下の表はマートパイプラインクラスにおける引数表である。
| 引数名 | 型|概要|
| :------: | :------- | :------- |
| `procs` | `Iterable[Literal["raw2inter","inter2primary","primary2primary"]]`|特定のプロセス「のみ」を実行したいときに使う。 |
|`to_procs`  |`Iterable[Literal["raw2inter","inter2primary","primary2primary"]]`| 特定のプロセス「まで」を実行したい場合に用いる。  |
|`from_procs` |`Iterable[Literal["raw2inter","inter2primary","primary2primary"]]`|特定のプロセス「以降」を実行したい場合に用いる。   |
|`to_nodes`  |`Iterable(str)`|特定のノード「まで」を実行したい場合に用いる。   |
|`from_nodes` |`Iterable(str)`|特定のノード「以降」を実行したい場合に用いる   |
|`node_names` | `Iterable(str)`|特定のノードを実行したい場合に用いる。  |
|`from_input` | `Iterable(str)`|特定のインプット以降のノードを実行したい場合に用いる。 |
|`to_outputs` | `Iterable(str)`|特定のアウトプットまでのノードを実行したい場合に用いる  |

### procs
`procs`引数を指定すると，渡された値に応じた**プロセスのみ**をもつパイプラインインスタンスを作成する。

以下にサンプルコードを示す。
1. `ModelPipeline`のみを実行するMLパイプラインインスタンスを作成
2. `raw2inter`のみを実行するマートパイプラインインスタンスを作成
```python
ml_pipeline.make(tags=['train'],procs=["model"]) # 1
mart_pipeline.make(procs=["raw2inter"]) # 2
```
このコード実行後にそれぞれで`run`コマンドを実行すると指定した値のみを実行できる。
### to_procs
`to_procs`引数を指定すると，渡された値**までのプロセス**をもつパイプラインインスタンスを作成する。

以下にサンプルコードを示す。
1. `TargetUserPipeline`，`FeaturePipeline`，`LabelPipeline`を実行するMLパイプラインインスタンスを作成
2. `raw2inter`,`inter2primary`を実行するマートパイプラインインスタンスを作成

```pythonAA
ml_pipeline.make(tags=['train'],to_procs=["label"])# 1
mart_pipeline.make(procs=["inter2primary"])# 2
```
このコード実行後にそれぞれで`run`コマンドを実行すると指定した値までのプロセスを実行できる。

### from_procs
`from_procs`引数を指定すると，渡された値**以降のプロセス**をもつパイプラインインスタンスを作成する。

以下にサンプルコードを示す。
1. `ModelPipeline`，`PostPipeline`を実行するMLパイプラインインスタンスを作成
2. `inter2primary`,`primary2primary`を実行するマートパイプラインインスタンスを作成
```python
ml_pipeline.make(tags=['train'],from_procs=["model"])# 1
mart_pipeline.make(procs=["inter2primary"])# 2
```
このコード実行後にそれぞれで`run`コマンドを実行すると指定した値以降のプロセスを実行できる。
### to_nodes
- `to_nodes`引数を指定すると，渡された値**までのノード**をもつパイプラインインスタンスを作成する。
- 指定できる値は定義したノード名。procsとは異なり固定値ではないので注意。
- ノード名の確認方法は`node_names`,`nodes`,`show`メソッドを実行することで可能。

以下にサンプルコードを示す。
```python
ml_pipeline.make(
    tags=['train'],
    to_nodes=["join_data([iris_targets_processed_train,iris_features_processed_train]) -> [feature_joined_data_train])"])
```
### from_nodes
- `from_nodes`引数を指定すると，渡された値**以降のノード**をもつパイプラインインスタンスを作成する。
- 指定できる値は定義したノード名。procsとは異なり固定値ではないので注意。
- ノード名の確認方法は`node_names`,`nodes`,`show`メソッドを実行することで可能。

以下にサンプルコードを示す。
```python
ml_pipeline.make(
    tags=['train'],
    from_nodes=["join_data([iris_targets_processed_train,iris_features_processed_train]) -> [feature_joined_data_train])"])
```

### node_names
- `node_names`引数を指定すると，渡された値**特定のノードのみ**をもつパイプラインインスタンスを作成する。
- 指定できる値は定義したノード名。procsとは異なり固定値ではないので注意。
- ノード名の確認方法は`node_names`,`nodes`,`show`メソッドを実行することで可能。

以下にサンプルコードを示す。
```python
ml_pipeline.make(
    tags=['train'],
    node_names=["join_data([iris_targets_processed_train,iris_features_processed_train]) -> [feature_joined_data_train])"])
```
### from_inputs
- `from_inputs`引数を指定すると，渡された値**特定のインプット以降**をもつパイプラインインスタンスを作成する。
- 指定できる値は定義したインプット名。nodes引数と同様に固定値ではないので注意。
- インプットの確認方法は`inputs`メソッドを実行することで可能。

以下にサンプルコードを示す。
```python
ml_pipeline.make(
    tags=['train'],
    from_inputs=["iris_features_processed_train"])
```
### to_outputs
- `to_outputs`引数を指定すると，渡された値**特定のインプット以降**をもつパイプラインインスタンスを作成する。
- 指定できる値は定義したアウトプット名。nodes引数と同様に固定値ではないので注意。
- アウトプットの確認方法は`output`メソッドを実行することで可能。

以下にサンプルコードを示す。
```python
ml_pipeline.make(
    tags=['train'],
    to_outputs=["iris_features_processed_train"])
```

## 遭遇しがちなエラー
下記のエラーは以下の可能性が高い。
- ValueError: Pipeline does not contain nodes named ・・・・
  - 指定したノード名がパイプラインにないため起こっている。
    - 引数の名前が間違っているもしくは定義し忘れの可能性が高い。
    - ”・・・” にアルファベットや記号が一つずつ区切られて表示されている場合は引数の値がIterableになっていない。
  
