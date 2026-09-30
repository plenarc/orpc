# oRPCプロジェクト概要

**状態:** レビュー対象のプロジェクト方針
**初期対象:** Nim 2.2.x
**ライセンス:** MIT

## 目的

oRPCは、Nim applicationにend-to-end type-safeなRPC開発体験を提供することを
目指します。最初からfull-stack frameworkを作るのではなく、小さくWeb framework
非依存なRPC layerから始めます。Nimで記述したcontractを唯一の情報源として、
application開発者が同じ型を二重管理することなく、server bindingとTypeScript
clientを導出できる状態を目標とします。

Nimにはnative codeの性能、静的型、読み書きしやすい構文、C/C++相互運用、
JavaScript compilation、compile-time metaprogrammingという強みがあります。
oRPCが最初に補うのは新しいWeb frameworkではなく、Nim serviceと一般的なWeb
frontendを接続できる、modernで長期運用可能なcontract境界です。

## プロダクト原則

1. **Framework agnostic:** coreはMummy、Prologue、HappyXおよびfrontend
   frameworkに依存しません。
2. **Contract first:** レビュー済みのNim宣言でoperationと型を定義し、transportと
   generated clientは正規化済みmetadataを利用します。
3. **End-to-end type safety:** 各targetで表現できる範囲までcontractの型を維持します。
   ただし、信頼できないwire dataにはruntime validationも必要です。
4. **Open standardsとlow lock-in:** 通常のHTTPとJSONを、文書化したsemanticsで
   利用します。metadataは将来OpenAPI 3.1を生成できる構造にし、標準的なHTTP
   toolingへ移行できる経路を残します。
5. **Nim-nativeで明示的なAPI:** compile-time metaprogrammingは活用しますが、
   予想外のglobal behavior、不透明なDSL、debug困難なmagicは避けます。
6. **Minimal dependencies:** contract/metadata layerは小さく保ち、optionalな連携は
   adapterまたは別packageへ分離します。
7. **商用利用に耐える方向性:** 機能数より、決定的な生成、互換性方針、有用な
   diagnostic、テスト、protocol behaviorの文書化を優先します。
8. **Public APIは人間がレビュー:** 提案文書のexampleは確定事項ではありません。
   public APIおよび大きなarchitecture変更はmaintainerの承認を必要とします。

## スコープ

### 最初の縦切り（v0.1候補）

最小の有用なflowは次のとおりです。

```text
型付きNim RPC宣言
        -> 正規化されたRPC metadata
        -> transport非依存の呼び出し + HTTP/JSON mapping
        -> 1つのframework adapter（Mummyが候補）
        -> framework非依存のgenerated TypeScript client
```

v0.1の目標は意図的に狭くします。限定したJSON互換input/output型、明示的な
registration、unary request/response、予測可能なerror、決定的なTypeScript生成を
対象とします。publicな宣言構文とwire formatの確定はレビュー項目として残します。

### 後から追加するが、設計時に考慮するもの

- Nim client生成、またはtyped Nim client
- OpenAPI 3.1出力
- より豊富なvalidation constraintとtyped application error
- Prologue、HappyX、その他server向けadapter
- versioningおよび互換性解析
- streamingまたはJSON以外のtransport（unary HTTP contract安定後）

### 現時点で明示的に対象外とするもの

- full-stack application framework
- Nim frontend framework
- core内のReact/Vue/Svelte/Solid固有binding
- Signal、Computed、Effect、Batch、Resource、Storeなどのfine-grained
  reactivity primitive
- authentication、persistence、dependency injection、deployment orchestration

React、Vue、Svelte、SolidなどのWeb applicationは、当面、同じ軽量なgenerated
TypeScript clientを利用します。

## 依存方向

依存は内側へ向けます。

```text
application
  -> oRPC framework adapter（例: Mummy）
       -> HTTP/JSON bridge
            -> dispatcher / 正規化されたcontract metadata

TypeScript generator -> 正規化されたcontract metadata
OpenAPI generator（将来） -> 正規化されたcontract metadata
```

coreからframework adapterをimportしません。adapterは各framework固有のrequest/
responseをoRPCの小さなHTTP abstractionへ変換し、host frameworkにrouteを公開します。
この規則により、contractを再定義せずにMummyを置き換えられます。

## 最初のreleaseの成功条件

- 小さなexampleで、1つのtyped RPCを宣言、実装、登録、呼び出しできる。
- 同じ正規化contractからTypeScript型とclient callを決定的に生成できる。
- 不正JSON、不正input、未知operation、handler failureのbehaviorが文書化・テストされ、
  機密情報を漏らさない。
- Web frameworkをinstallせずにcoreおよびgeneratorのテストを実行できる。
- 少なくとも1つのadapterで境界を実証し、その依存をcoreへ追加しない。
- 制限事項と互換性の期待値を明記する。

## 長期的な方向性

oRPCは将来、Nim backendとmodern frameworkのfine-grained reactivityを参考にした
Nim frontendの中間層になる可能性があります。ただし、この可能性によって現在の
RPC境界を歪めてはいけません。TypeScript clientと標準的なHTTP interfaceは、
一時的な足場ではなくfirst-class productです。
