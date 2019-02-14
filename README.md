# Ayame Vue サンプル

このサンプルは [WebRTC Signaling Server Ayame](https://github.com/shiguredo/ayame) で利用可能な Vue を利用したサンプルです。

## 動かし方

[Ayame Server](https://github.com/shiguredo/ayame) をクローンし、USE.md を参考に起動しておきます。

このリポジトリをクローンし、

```
$ yarn install
```
を実行し、依存ライブラリをインストールしたら、

```
$ yarn  serve
```

を実行して、クライアントを serve します。

```
 DONE  Compiled successfully in 146ms

Type checking and linting in progress...

  App running at:
  - Local:   http://localhost:8080/
  - Network: http://192.168.11.53:8080/
```
- http://localhost:8080/ にアクセスすると、起動しているアプリが表示されます。

- (オプション) input form にシグナリングサーバの向き先を指定します。デフォルトだと ws://localhost:3000/ws  になっています。

- 「接続」をクリックすると、P2P 接続が開始します。


## ライセンス

Apache License 2.0

```
Copyright 2019, Shiguredo Inc, kdxu

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
