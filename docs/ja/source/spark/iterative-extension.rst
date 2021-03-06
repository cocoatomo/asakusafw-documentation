=====================================
Asakusa on Spark Iterative Extensions
=====================================

この文書では、Asakusa on Sparkの拡張機能「Iterative Extensions」について説明します。

..  attention::
    本ドキュメントのバージョンでは、 Iterative Extensions は試験的機能として提供しています。

概要
====

Iterative Extensionsは、あるバッチに対してバッチ引数の一部または全部を変えながら同じバッチを連続して実行するための機能です。

Iterative Extensionsを適用したバッチを「反復バッチ」と呼びます。
反復バッチは通常のバッチを連続して実行する場合と比べて、次の点で高速に実行できる可能性があります。

* 連続処理によるリソースの効率的な利用

 連続するバッチアプリケーションを1つのSparkアプリケーションとして実行するため、特にYARN上での実行においては、アプリケーションコンテナの初期化などの分散オーバーヘッドが極小化される、コンテナリソースをシンプルな設定で最大限に利用できる、などの利点があります。

* 差分処理による最適化

 反復バッチでは連続するバッチ間で再計算が不要な箇所は実行結果を再利用することがあるため、特に実行するバッチアプリケーション間での変更箇所が少ない場合には、バッチ間の差分処理による利点が大きくなります。

反復バッチは、日付範囲を指定した日次バッチの一括実行や、パラメータ・スイープによるシミュレーションといった用途に適しています。

Iterative Extensionsは、反復バッチを定義するためのAsakusa DSLの拡張構文、反復バッチを生成するするためのAsakusa DSLコンパイラの拡張、および反復バッチを実行するためのインターフェースや実行モジュールなどを提供します。

..  attention::
    現時点では、Iterative Extensions は Asakusa on Spark 上でのみ使用できます。
    MapReduce実行環境に対してIterative Extensionsに対応する予定はありません。

反復バッチの開発
================

反復バッチを作成するには、 `Iterative Extensions用の構文`_ を利用して、 `反復バッチの定義`_ 、`反復演算の定義`_ を行います。

反復バッチ固有の定義以外は、通常のAsakusa DSLを使ったバッチアプリケーションの開発方法と同じです。

Iterative Extensions用の構文
----------------------------

Asakusa Framework バージョン ``0.8.0`` 以降では、Asakusa DSLに対して以下の構文を利用することができます。

反復バッチ注釈 - ``@IterativeBatch`` [#]_
  対象の **ジョブフロー** が反復バッチであることを表す注釈。

  ジョブフロークラスに反復バッチ注釈 ``IterativeBatch`` を指定し、アプリケーションのDSLコンパイルを実行すると該当のジョブフローから反復バッチを生成します。

  ..  attention::
      反復バッチ注釈はバッチクラスではなく、ジョブフロークラスに指定します。
      反復バッチでは Batch DSL は使用しません。

反復化注釈 - ``@Iterative`` [#]_
  対象の処理 (外部入力、演算子) が反復対象であることを表す注釈。

  ジョブフロークラスのジョブフローコンストラクタや演算子クラス、演算子メソッドに反復化注釈 ``Iterative`` を指定することで、その処理が反復処理の対象となります。

  反復化注釈には、その要素に反復の対象となる変数を表す反復変数名を指定することができます。

..  [#] :asakusafw-javadoc:`com.asakusafw.vocabulary.iterative.IterativeBatch`
..  [#] :asakusafw-javadoc:`com.asakusafw.vocabulary.iterative.Iterative`

反復バッチの定義
----------------

反復バッチを定義するには、ジョブフロークラスに対して反復バッチ注釈 ``IterativeBatch`` を指定します。
この注釈の要素 ``name`` に反復バッチの名前を指定します。この名前は反復バッチのバッチIDとして使用されます [#]_ 。

..  code-block:: java

    @IterativeBatch(name = "hogeIterativeBatch")
    public class HogeJobFlow extends FlowDescription {
        ...
    }

既存のジョブフロークラス ( 注釈 ``JobFlow`` を指定したクラス ) に反復バッチ注釈を指定することもできます。
この場合、このジョブフロークラスは通常のバッチにおけるジョブフロー、および反復バッチの両方の要素として利用することができます。

..  code-block:: java

    @JobFlow(name = "hogeJobflow")
    @IterativeBatch(name = "hogeIterativeBatch")
    public class HogeJobFlow extends FlowDescription {
        ...
    }

..  [#] 反復バッチのフローIDには自動的に ``main`` という値が設定されます。

反復演算の定義
--------------

反復バッチの中で反復対象となる箇所を「反復演算」として定義します。

反復演算には、 `外部入力の反復`_ と `演算子の反復`_ を定義することができます。

また、反復演算の指定時に `反復変数の指定`_ を行うことで、繰り返し処理時に不必要な再処理を行わず、演算処理の結果を再利用するといった最適化が得られる可能性があります。

反復演算を含むデータフローは、 `反復演算の対象範囲`_ に従って反復処理が行われます。

外部入力の反復
~~~~~~~~~~~~~~

外部入力の反復は、Direct I/O を使った外部入力処理に対して、バッチ引数を変更しつつ連続して入力処理を実行するよう指定します。
外部入力の反復は主に入力の対象や範囲をパラメータによって切り替えながら実行する、といった場合に利用します。

例えばDirect I/Oでは、入力ファイルのベースパスやファイル名のパターン文字列の一部などをバッチ引数の値で置き換えることができますが、反復バッチによってこれらのパラメータを変更しつつ連続で入力処理を実行できるようになります [#]_ 。

外部入力の反復を指定するには、 `反復バッチの定義`_ で定義したジョブフロークラスのジョブフローコンストラクタ内で、反復演算の対象とする入力（ 注釈 ``Import`` を指定している仮引数）の先頭に反復化注釈 ``Iterative`` を指定します。

..  code-block:: java

    @IterativeBatch(name = "hogeIterativeBatch")
    public class HogeJobFlow extends FlowDescription {
        ...
        public HogeJobFlow(
                @Iterative @Import(name = "foo", description = "FooImporter.class") In<Foo> input,
                ...) {
            ...
        }
        ...
    }

..  attention::
    現時点では、WindGate, ThunderGateによる外部入力はIterative Extensionsに対応していません。
    これらの外部入力に反復化注釈を設定した場合はDSLコンパイルエラーとなります。

..  attention::
    Interative ExtensionsはSparkの実行環境にのみ対応しています。
    このため外部入力の反復を利用する場合は、コンパイルオプション ``spark.input.direct`` [#]_ を ``false`` に設定した、
    MapReduce上でDirect I/Oの入力処理を実行する機能は利用できません。
    コンパイルオプション ``spark.input.direct`` を ``false`` に設定した場合はDSLコンパイルエラーとなります。

..  [#] Direct I/O の入力時にバッチ引数が利用可能な項目については :doc:`../directio/user-guide` などを参照してください。
..  [#] コンパイルオプション ``spark.input.direct`` については、 :doc:`reference` のコンパイラプロパティの項を参照してください。

演算子の反復
~~~~~~~~~~~~

演算子の反復は、バッチ引数を利用する処理を記述したユーザ演算子に対して、バッチ引数を変更しつつ連続して演算子の処理を実行するよう指定します。外部入力の反復は主に入力データを切り替えるのに対して、演算子の反復は演算子内のロジックで使用するパラメータを切り替える場合に利用します。

ユーザ演算子の演算子メソッド内ではコンテキストAPIを使ってバッチ引数を取得することができますが、反復バッチによってコンテキストAPIから取得するバッチ引数の値を変更しつつ連続で演算子の処理を実行することができるようになります [#]_ 。

演算子の反復を指定するには、反復演算の対象とするユーザ演算子に対して、演算子注釈の前に反復化注釈 ``Iterative`` を指定します。

..  code-block:: java

    public abstract class HogeOperators {
        ...
        @Iterative
        @Update
        public void hogeOperator(Bar bar) {
            String iterativeParameter = BatchContext.get("<iterative-parameter-key>");
            ...
        }
        ...
    }

..  [#] コンテキストAPIの使い方については :doc:`../dsl/user-guide` などを参照してください。

反復変数の指定
~~~~~~~~~~~~~~

バッチ引数のうち、反復バッチによって連続処理の都度変更の対象となるバッチ引数を「反復変数」と呼びます。
通常のバッチでは複数のバッチ引数を指定できるのと同様に、反復バッチでは複数の反復変数を設定することができます。

反復バッチ内のある反復演算内では、反復バッチに与えた反復変数に対して一部の反復変数のみを利用する場合があります。
そのような反復演算については、反復演算の定義時にその処理内で利用する反復変数を指定しておくことで、不要な再処理を実行しないような最適化が得られる可能性があります。

反復演算に対して反復変数を指定するには、反復化注釈 ``Iterative`` の要素に反復変数名を指定します。
反復変数名は複数指定が可能です。

..  code-block:: java

    public abstract class HogeOperators {
        ...
        @Iterative({ "iterative-param1", "iterative-param2" })
        @Update
        public void hogeOperator(Bar bar) {
            String iterativeParameter1 = BatchContext.get("iterative-param1");
            String iterativeParameter2 = BatchContext.get("iterative-param2");
            ...
        }
        ...
    }

なお、反復化注釈に反復変数を設定しない場合は、その反復演算は連続処理の都度、常に再処理が必要であるものとして扱われます。

反復演算の対象範囲
~~~~~~~~~~~~~~~~~~

データフロー内で、ある反復演算に後続する演算子の処理は自動的に反復演算となります。

このような演算子内では、明示的に反復演算の指定を行なわなくても反復変数を利用することができます。

外部出力の反復
~~~~~~~~~~~~~~

外部出力の反復は、Direct I/O を使った外部出力処理に対して、バッチ引数を変更しつつ連続して出力処理を実行します。

外部出力に接続されるデータフロー内で外部入力の反復や反復演算が実行された場合、
外部出力の入力ファイルのベースパスやファイル名のパターン文字列の一部などに反復変数を設定していると、これを反復演算として処理します。

なお、外部出力は明示的に反復演算として定義することはできません。外部入力と同様の方法で反復化注釈を外部出力に指した場合はDSLコンパイルエラーとなります。

..  attention::
    現時点では、WindGate, ThunderGateによる外部出力はIterative Extensionsに対応していません。
    これらの外部出力を利用する場合、外部入力と外部出力で同じバッチ引数を使用している場合において外部入力を反復演算としても、外部出力側では反復化の対象とはならないことに注意してください。

..  attention::
    Interative ExtensionsはSparkの実行環境にのみ対応しています。
    このため外部出力の反復を利用する場合は、コンパイルオプション ``spark.output.direct`` [#]_ を ``false`` に設定した、
    MapReduce上でDirect I/Oの出力処理を実行する機能は利用できません。

..  note::
    Asakusa on Spark バージョン 0.3系では、外部出力の反復はIterative Extensionsに対応していませんでした。

    Asakusa on Spark バージョン 0.4.0 からは上述の通り、Direct I/Oを外部出力に利用する場合においてIterative Extensionsに対応するようになりました。

..  [#] コンパイルオプション ``spark.output.direct`` については、 :doc:`reference` のコンパイラプロパティの項を参照してください。

反復バッチのテスト
------------------

現時点では、Iterative Extensionsでは反復バッチ特有のテスト機能は提供していません。

反復バッチをテストする方法として、反復バッチのジョブフロークラスを通常のジョブフローとしてテストする方法があります。
テストドライバ上から実行する場合、パラメータの反復をテスト上で再現することはできませんが、単一のパラメータセットに対してのテストは可能です。

..  tip::
    反復バッチ専用のジョブフロークラスを作成した場合、テストドライバ上でテストを実行するのみの目的で 注釈 ``JobFlow`` を付与するのは望ましくないかもしれません。その場合、 ``FlowPartTester`` を使って対象のジョブフロークラスをフロー部品としてテストを実行する方法があります。

反復バッチのビルド
------------------

反復バッチのビルド方法は通常のバッチアプリケーションのビルド手順と同じです。

ビルド用のGradleタスクとして :program:`sparkCompileBatchapps` や :program:`assemble` を利用することができます。
これらのタスクを実行すると、アプリケーションプロジェクトの :file:`build/spark-batchapps` 配下にビルド済みのバッチアプリケーションが生成されます。

なお、Asakusa on Spark Gradle Pluginを有効にしている場合、 :program:`assemble` タスクによるデプロイメントアーカイブの作成時に反復バッチの実行に必要なモジュールが含まれるため、
追加のライブラリ登録などは必要ありません。

アプリケーションのビルドやデプロイについては、 :doc:`user-guide` も参照してください。

反復バッチの実行
================

反復バッチは通常のバッチと同様にYAESSを使って実行することができますが、反復バッチ固有のパラメータが必要です。

変数表の作成
------------

反復バッチ実行時に指定する反復変数の一覧を「変数表」と呼びます。

変数表は、JSON形式のUTF-8テキストファイルとして定義します。
1つのJSONオブジェクトに1回分のバッチ実行処理に必要なバッチ引数をプロパティとして定義します。

この1つのJSONオブジェクトで定義する、反復変数の一覧を一意に定めた1回分の処理を「ラウンド」と呼びます。
変数表には、反復バッチ内の各ラウンドで使用する反復変数を定義したJSONオブジェクトを列挙します。

次の変数表の例では、反復変数 ``date`` を3ラウンド分定義しています。

..  code-block:: json

    {
        "date": "2011-04-01"
    }
    {
        "date": "2011-04-02"
    }
    {
        "date": "2011-04-03"
    }

1ラウンド内で複数の反復変数を指定する場合は、次の例のように定義します。
ここでは 反復変数 ``date`` と ``category`` に対してそれぞれ2つの値の組み合わせ、つまり4つのパターンを各ラウンドで実行します。

..  code-block:: json

    {
        "date": "2011-04-01",
        "category": "01"
    }
    {
        "date": "2011-04-01",
        "category": "02"
    }
    {
        "date": "2011-04-02",
        "category": "01"
    }
    {
        "date": "2011-04-02",
        "category": "02"
    }

..  attention::
    変数表のJSONファイルはJSONの配列ではなく、JSONのオブジェクトを列挙した形で指定してください。
    オブジェクトの区切りにカンマ等も不要です。

YAESSによる反復バッチの実行
---------------------------

反復バッチを実行するには、YAESSのオプションに `変数表の作成`_ で用意した変数表を指定します。

``yaess-batch.sh`` のオプションに ``-X-parameter-table <変数表のファイルパス>`` という形式で変数表のファイルパスを指定することができます。

``-X-parameter-table`` による変数表の指定と、``-A <変数名>=<値>`` によるバッチ引数の指定を同時に行うこともできます。
変数表内のバッチ引数と、 ``-A`` で指定するバッチ引数で同じ変数名の指定が存在した場合、変数表で指定する値が使用されます。

以下は、YAESSによる反復バッチの実行例です。

..  code-block:: sh

    $ASAKUSA_HOME/yaess/bin/yaess-batch.sh hogeIterativeBatch -X-parameter-table $HOME/var/parameter-table.json

通常のバッチ引数と変数表を両方指定する場合は、以下のように指定します。

..  code-block:: sh

    $ASAKUSA_HOME/yaess/bin/yaess-batch.sh hogeIterativeBatch -A foo=abc -X-parameter-table $HOME/var/parameter-table.json


反復バッチの実行時設定
----------------------

反復バッチの実行時パラメータは、 :doc:`optimization` と同じ方法で設定することができます。

以下では反復バッチ固有の設定項目について説明します。

設定項目
~~~~~~~~

``com.asakusafw.spark.iterativebatch.slots``
  反復バッチ内で同時に実行するラウンド数を指定します。

  このプロパティを設定しない場合、反復バッチの実行時にすべてのラウンドを同時に実行します。

  既定値: ``Integer.MAX_VALUE``

  ..  hint::
      一部のケースにおいて、同時に実行するラウンド数が大きい場合にタスク数が膨大になることで、Sparkアプリケーションのパフォーマンスが劣化することがあることを確認しています。
      このような場合、``com.asakusafw.spark.iterativebatch.slots`` を適切に設定することでパフォーマンスが改善する可能性があります。

``com.asakusafw.spark.iterativebatch.stopOnFail``
  反復バッチ実行中のあるラウンドが異常終了した場合に、反復バッチ全体を異常終了するかを指定します。

  標準の設定では、反復バッチ内であるラウンドが異常終了した場合は即時に反復バッチ全体を異常終了します。

  この設定値を ``false`` にした場合、あるラウンドが異常終了しても他のラウンドの処理が続行されます。また反復バッチの実行結果 （正確には反復バッチ内の ``main`` フェーズ）は常に成功となります。

  既定値: ``true``

  ..  attention::
      この設定値を ``false`` にした場合、一部、もしくは全てのラウンドが異常終了した場合でも、反復バッチの実行結果が成功となることに注意してください。
      各ラウンドの実行結果は、反復バッチの実行時ログなどを確認する必要があります。
