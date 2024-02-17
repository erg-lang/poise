# アーキテクチャ

The Erg package manager (コードネーム: poise)はErgを用いて実装されている。現状では実用可能なErgのバックエンドがCPythonバックエンドしかないため、poiseは単体のバイナリとしては提供されておらず、`erg pack`サブコマンドが内部的に呼び出すアプリケーションパッケージとなっている。将来的にErgにネイティブコードバイナリが追加された場合でもパッケージ管理は基本的に`erg pack`で行う。

poiseは以下のコマンドを持つ。

* `init`: パッケージを初期化する
* `install`: パッケージをインストールする(`erg install`と同じ、パッケージの依存関係を追加する場合は`add`)
* `update`: パッケージをアップデートする
* `add`: パッケージの依存関係を追加する
* `clean`: パッケージをクリーニングする(キャッシュの削除など)
* `build`: パッケージをビルドする(`erg build`と同じ)
* `run`: パッケージをビルドし、実行する
* `publish`: packages.erg-lang.orgにパッケージを公開する
* `test`: パッケージのテストを実行する(`erg test`と同じ)

各サブコマンドの内部処理について解説する。

## init

規定のディレクトリ構成をセットアップする。規定のディレクトリ構成は以下の通り[^1]。

```console
/package # package root directory, this is also the package name
    /build # Directory to store build results
        /debug # Artifacts during debug build
        /release # Artifacts of release build
    /doc # Documents (in addition, by dividing into subdirectories such as `en`, `ja` etc., it is possible to correspond to each language)
    /src # source code
        /main.er # file that defines the main function
    /tests # Directory to store (black box) test files
    /package.er # file that defines package settings
```

initの仕事はbuildを除くディレクトリを自動で作成することである。`--app`と`--lib`はそれぞれアプリケーションパッケージとライブラリパッケージを作成する。`--app`を指定した場合、`src`以下に`main.er`が作成される。`--lib`を指定した場合、`src/lib.er`が作成される。両方指定することも可能であり、`src/main.er`と`src/lib.er`が作成される。

## install

アプリケーションパッケージをインストールする。パッケージ名を指定しなかった場合、`package.er`に記述された依存関係をインストールする。パッケージは`.erg`以下にキャッシュされ、他のパッケージでキャッシュが再利用される場合がある。

アプリケーションバッケージのインストールについて内部処理のフローを説明する。

1. `erg-lang/package-index`からパッケージのメタデータを取得する

* indexはcargoに倣いsparse registryとして実装されている。すなわち、全てのパッケージ情報を単一のjsonなどで管理するのではなく、名前ごとに複数のディレクトリに分割し、そのディレクトリ内にパッケージのメタデータをjsonなどで管理する。パッケージのメタデータはjson形式である。

  * 例:
    * `erg`: `package-index/certified/e/erg.json`
    * `foo-bar`: `package-index/certified/f/foo-bar.json`
    * `foo/bar`: `package-index/developers/foo/b/bar.json`

ergのパッケージレジストリはまず開発者`developers`ごとに名前空間が別れていることに注意されたい。その後リクエストのあったパッケージは審査を経て`certified`にも登録される。
パッケージをインストールする際に開発者名が指定されなかった場合`certified`名前空間から検索されることになる。見つからなかった場合、各開発者の名前空間から検索される。

jsonの中身は以下のようにバージョンごとに整列されている。つまり、`package.er`から`name`と`version`を抜き、その他の情報がjsonにシリアライズされて配置される。
大小関係の判定はsemverに従う。

```json
{
    "versions": {
        "0.1.0": {
            "description": "an awesome package",
            "dependencies": {
                "foo": "0.1.0"
            },
            ...
        }
    },
    ...
}
```

2. リソースのダウンロード

後述するようにパッケージはキャッシュされるので、既に同一バージョンの同一パッケージがダウンロードされている場合このステップは省かれる。

特にバージョンが指定されなかった場合、json内の一番下のバージョンがインストールされる。

Erg package systemでは、再現性のためパッケージは全て圧縮されてindex内に保存される。圧縮形式はtar.gzであり、jsonと同じディレクトリに配置される。例えば、`erg`の場合は`package-index/certified/e/erg/0.1.0.tar.gz`となる。

さらにjson内に記述されているdependenciesから再帰的に依存関係を解決し、必要なパッケージをダウンロードする。

ダウンロードされたパッケージは解凍されて`.erg`以下に配置される。例えば、`erg`の場合は`.erg/packages/github.com/certified/erg/0.1.0`となる。`foo/bar`の場合は`.erg/packages/github.com/developers/foo/bar/0.1.0`となる。

複数のモジュールが同一のパッケージの別バージョンを利用している場合、新しい方のバージョンのみを用いることができないかトライされる。これに失敗しても別バージョンが追加でインストールされるだけでビルドは継続される。この依存関係解決の結果は後述するpackage.lock.erに保存される。

パッケージをどのように読み込みリンクするかはコンパイラの責務となる。具体的には、コンパイラはpackage.erがプロジェクトルートにある場合、その中のdependenciesをプロジェクト内で使えるパッケージとして認識する。実際の名前解決には後述するpackage.lock.erを用いる。

3. ロックファイルの生成

これは2.と並行して行われる。パッケージのバージョンはsemantic versioningに従って範囲指定することができる。そしてパッケージは日々アップデートされるので、package.erの情報だけでは再現性が担保されない。そこで、パッケージのバージョンを固定するためにロックファイルが用意されている。ロックファイルは`package.lock.er`という名前で`package.er`と同一のディレクトリに配置される。

例:

```erg
.packages = {
    .foo = { .version = "0.1.0", features = ["debug"] },
    .bar = { .version = "0.1.1" },
    ...
}
```

ビルド時にコンパイラはpackage.lock.erを見ながらパッケージを名前解決する。package.lock.erは手動で編集することも可能であるが、コンパイラはパッケージ管理の責務を負わないので編集後にergcを用いてコンパイルすると名前解決に失敗する可能性がある。`erg pack build`/`erg build`ならば毎回package.lock.erの検証を行うので安全である。

## update

アプリケーションパッケージをアップデートする。パッケージ名を指定しなかった場合、`package.er`に記述された依存関係をアップデートする。

依存関係のアップデートは貪欲(greedy)に行われる。例えばパッケージAとBがCに依存していて、Aの場合はCのバージョンアップが可能だがBの場合は不可能な場合、Aのみがアップデートされる。従って複数のパッケージ間で汎用的に共用されるパッケージは多数のバージョンが内部的に併存する場合がある。

## build

パッケージをビルドする。poiseはpackage.lock.erを検証し、パスすればコンパイラにコンパイルを命令する。
コンパイルの成果物(.pycファイルまたはネイティブコード)はbuild以下に配置される。デフォルトではdebugビルドが行われ、成果物は`build/debug`以下に出力されるが、`--release`を指定することでreleaseビルドが行われ、`build/release`以下に出力される。これはpackage.erのあるプロジェクトの場合コンパイラが配置する。

現在は未実装だが、コンパイラがインクリメンタルビルドに対応した場合、`build/{debug, release}`以下にビルド成果物がキャッシュされる。

ビルド時はtests、examples以下のファイルも検査される。またファイル内のdoc commentsもergコードブロックがあれば検査される。

---

[^1]: パッケージ内で利用されるサブパッケージは`packs`以下に配置することが推奨される。しかしErgの場合モジュール=1ファイル単位でキャッシュ&並列コンパイルされるのでRustほどパッケージ(crate)を分割するメリットはない。
