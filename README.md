# Okapi — マルチテナントAPIゲートウェイ
=================================

Copyright (C) 2017 The Open Library Foundation

このソフトウェアはApache Licenseの条項の下で配布され、
バージョン2.0です。詳細は、 "[LICENSE](LICENSE)" ファイルを参照してください。

システム要求
-------------------

TOkapiソフトウェアには、以下のコンパイル時の依存関係があります:

* Java 8

* Apache Maven 3.3.x 以上

さらに、テストスイートは成功するためにポート9130-9134にバインドできる必要があります。

*注：テストが失敗した場合、API Gatewayがそれが生まれたマイクロサービスをシャットダウンできない場合があります
。その場合、手動で終了する必要があるかもしれません*

クイック・スタート
-----------

ビルドして実行するには:

    $ mvn install
    $ mvn exec:exec

Okapi はポート9130をリスニングします。

ドキュメンテーション
-------------

* [Okapi Guide and Reference](doc/guide.md)
* [Contributing guidelines](CONTRIBUTING.md)
* 他のFOLIO開発ドキュメントは [dev.folio.org](http://dev.folio.org/)
* [FOLIO issue tracker](http://dev.folio.org/community/guide-issues)のproject [OKAPI](https://issues.folio.org/browse/OKAPI)をご覧ください。
