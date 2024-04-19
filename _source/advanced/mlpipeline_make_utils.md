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
下記の表に`make`コマンドに付帯する引数の一覧を記載する。なお全て任意である。

| 引数名 | 概要|
| :------: | :------- |
| `procs` | 特定のプロセス「のみ」を実行したいときに使う。 |
|`to_procs`  | 特定のプロセス「まで」を実行したい場合に用いる。  |
|`from_procs` |特定のプロセス「以降」を実行したい場合に用いる。   |
|`to_nodes`  |特定のノード「まで」を実行したい場合に用いる。   |
|`from_nodes` |特定のノード「以降」を実行したい場合に用いる   |
|`node_names` | 特定のノードを実行したい場合に用いる。  |
|`from_input` | 特定のインプット以降のノードを実行したい場合に用いる。 |
|`to_outputs` | 特定のアウトプットまでのノードを実行したい場合に用いる  |

### procs
`procs`引数を指定すると，渡された値に応じた**プロセスのみ**をもつパイプラインインスタンスを作成する。
引数に渡すことができる値はパイプラインによって変化する。

| パイプライン名 | 値|
| :------: | :------- |
| MLパイプライン | `pre`<br>`target`<br>`feature`<br>`label`<br>`join`<br>`model`<br>`post` | `Iterable(str)`|
|マートパイプライン  | `raw2inter`<br>`inter2primary`<br>`primary2primary` |`Iterable(str)`|

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
渡せる引数の値は`procs`と同じ。
| パイプライン名 | 値|
| :------: | :------- |
| MLパイプライン | `pre`<br>`target`<br>`feature`<br>`label`<br>`join`<br>`model`<br>`post` | `Iterable(str)`|
|マートパイプライン  | `raw2inter`<br>`inter2primary`<br>`primary2primary` |`Iterable(str)`|

以下にサンプルコードを示す。
1. `TargetUserPipeline`，`FeaturePipeline`，`LabelPipeline`を実行するMLパイプラインインスタンスを作成
2. `raw2inter`,`inter2primary`を実行するマートパイプラインインスタンスを作成

```python
ml_pipeline.make(tags=['train'],to_procs=["label"])# 1
mart_pipeline.make(procs=["inter2primary"])# 2
```
このコード実行後にそれぞれで`run`コマンドを実行すると指定した値までのプロセスを実行できる。

### from_procs
`from_procs`引数を指定すると，渡された値**以降のプロセス**をもつパイプラインインスタンスを作成する。
指定できる値は`to_procs`と同様。
| パイプライン名 | 値|
| :------: | :------- |
| MLパイプライン | `pre`<br>`target`<br>`feature`<br>`label`<br>`join`<br>`model`<br>`post` | `Iterable(str)`|
|マートパイプライン  | `raw2inter`<br>`inter2primary`<br>`primary2primary` |`Iterable(str)`|

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
- 指定できる値は定義したノード名。ノード名の確認方法は`node_names`,`nodes`,`show`メソッドを実行することで可能。

以下にサンプルコードを示す。
1. `ModelPipeline`，`PostPipeline`を実行するMLパイプラインインスタンスを作成
2. `inter2primary`,`primary2primary`を実行するマートパイプラインインスタンスを作成
```python
ml_pipeline.make(tags=['train'],to_nodes=[" "])# 1
mart_pipeline.make(procs=[""])# 2
```
### from_nodes
- `from_nodes`引数を指定すると，渡された値**以降のノード**をもつパイプラインインスタンスを作成する。
- 指定できる値は定義したノード名。ノード名の確認方法は`node_names`,`nodes`,`show`メソッドを実行することで可能。

以下にサンプルコードを示す。
1. `ModelPipeline`，`PostPipeline`を実行するMLパイプラインインスタンスを作成
2. `inter2primary`,`primary2primary`を実行するマートパイプラインインスタンスを作成
```python
ml_pipeline.make(
    tags=['train'],
    from_nodes=[""])# 1
mart_pipeline.make(procs=[""])# 2
```

### node_names
- `node_names`引数を指定すると，渡された値**特定のノードのみ**をもつパイプラインインスタンスを作成する。
- 指定できる値は定義したノード名。ノード名の確認方法は`node_names`,`nodes`,`show`メソッドを実行することで可能。

以下にサンプルコードを示す。
1. `ModelPipeline`，`PostPipeline`を実行するMLパイプラインインスタンスを作成
2. `inter2primary`,`primary2primary`を実行するマートパイプラインインスタンスを作成
```python
ml_pipeline.make(tags=['train'],node_names=[""])# 1
mart_pipeline.make(procs=[""])# 2
```

### from_input
- `node_names`引数を指定すると，渡された値**特定のインプット以降**をもつパイプラインインスタンスを作成する。
- 指定できる値は定義したノード名。ノード名の確認方法は`node_names`,`nodes`,`show`メソッドを実行することで可能。

以下にサンプルコードを示す。
1. `ModelPipeline`，`PostPipeline`を実行するMLパイプラインインスタンスを作成
2. `inter2primary`,`primary2primary`を実行するマートパイプラインインスタンスを作成
```python
ml_pipeline.make(tags=['train'],from_input=" ")# 1
mart_pipeline.make(procs="inter2primary")# 2
```
### to_outputs
|`to_outputs` | 特定のアウトプットまでのノードを実行したい場合に用いる  |
- `to_outputs`引数を指定すると，渡された値**特定のインプット以降**をもつパイプラインインスタンスを作成する。
- 指定できる値は定義したノード名。アウトプットの確認方法は`output`メソッドを実行することで可能。

以下にサンプルコードを示す。
1. `ModelPipeline`，`PostPipeline`を実行するMLパイプラインインスタンスを作成
2. `inter2primary`,`primary2primary`を実行するマートパイプラインインスタンスを作成
```python
ml_pipeline.make(tags=['train'],to_outputs=" ")# 1
mart_pipeline.make(procs="inter2primary")# 2
```

