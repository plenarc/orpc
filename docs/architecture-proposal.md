# oRPC v0.1 architecture案

**状態:** 提案。public APIおよびprotocol詳細はmaintainerレビューが必要
**関連文書:** [プロジェクト概要](project-brief.md)

## 1. 提案する責務境界

architectureを、個別にテスト可能な次のpipelineへ分割します。

1. **Contract capture（compile time）:** pragma macroまたはcontract macroが通常の
   Nim宣言を調べ、対応範囲に含まれるか検証します。
2. **正規化metadata:** capture処理はservice、procedure、parameter、result、error、
   serializable type shapeのcanonicalな記述を生成します。metadataにMummy objectや
   TypeScript固有文字列を含めません。
3. **Bindingとdispatch:** 明示的なregistrationでmetadataとhandler procを結びます。
   transport非依存dispatcherはoperation identifierとserialized requestを受け取り、
   successまたはprotocol errorを返します。
4. **HTTP/JSON bridge:** 小さな層でHTTP method/path/body/statusとJSONをdispatcherの
   request/responseへ対応付けます。
5. **Adapter:** 最初の候補であるMummyは、固有request/response APIとbridgeを変換
   します。他のadapterも同じ役割を独立して実装します。
6. **Generator:** TypeScript generatorは正規化modelを読み、安定した型と、交換可能な
   `fetch`互換transportを使うclientを生成します。OpenAPIおよびNim client generatorも
   将来同じmodelを利用できます。

metadataには用途の異なる2形態があり、混同しないようにします。

- binding生成と良いerror表示に使う、型付きのcompile-time representation
- generatorとsnapshot testに使う、単純でserialize可能なintermediate
  representation（IR）

その境界をNimで安定して受け渡せるかPoCで確認してから、formatをpublicにします。

## 2. Protocol候補（未確定）

最初はunary JSON callだけを対象とします。保守的な候補は、
`POST /rpc/<service>/<operation>`のようにoperationごとに安定したrouteを設け、named
parameterを持つJSON objectとJSON result envelopeを使う方式です。positional arrayより
named parameterのほうが安全に進化させやすく、validation errorも明確になります。

PoCでは、単純なHTTP success bodyと次のようなenvelopeを比較します。

```json
{ "ok": true, "data": { "id": 42, "name": "Ada" } }
```

failure候補は次のとおりです。

```json
{ "ok": false, "error": { "code": "NOT_FOUND", "message": "User not found" } }
```

上記の名称は確定仕様ではありません。確定前に、integerと64-bit value、float、enum、
optional/null、object、sequence、map、timestamp、binary data、unknown fieldのJSON表現を
定義します。HTTP status mapping、content-type、duplicate operation検出、exceptionの
sanitize、request size policy、cancellation、evolution ruleも定義が必要です。

typed errorは最終的に、安定したcodeとpayload schemaを持つdataとして宣言します。
想定外のdefectはinternal errorとし、stack traceをserializeしてはいけません。
transport error、protocol/validation error、宣言済みapplication error、想定外server
errorをclient側で区別できるようにします。

## 3. Repository/module構成案

当初は単一repositoryとしますが、依存の重さやrelease cycleによってadapterを後から
分離できるよう、package境界を維持します。

```text
orpc.nimble
src/
  orpc.nim                         # 小さく精選したpublic facade
  orpc/
    contract.nim                  # public annotation/宣言helper
    metadata.nim                  # canonicalなtransport非依存model
    types.nim                     # 対応type shape表現
    validation.nim                # metadata/wire validationとdiagnostic
    dispatch.nim                  # handler bindingと呼び出し
    protocol.nim                  # protocol result/error vocabulary
    http_json.nim                 # framework非依存HTTP/JSON mapping
    codegen/
      typescript.nim              # 決定的TS AST/rendering entry point
      model.nim                   # generator向けserialize可能IR
  orpc_mummy.nim                  # optionalなpublic Mummy adapter facade
  orpc/
    adapters/
      mummy.nim                   # Mummyをimport。coreからはimportしない
tests/
  compile/                        # accept/rejectするcontract fixture
  unit/                           # metadata、validation、protocol、dispatcher
  integration/                    # adapter + HTTP round trip
  golden/                         # 決定的なgenerated TS fixture
examples/
  mummy_typescript/               # API承認後の最小vertical slice
docs/
```

1つのpackageではMummyを真にoptionalにできない場合、すべてのcore userにinstallさせる
のではなく、adapterを`orpc_mummy`などのsibling packageとして公開します。これは推測で
決めず、PoCで検証します。generated TypeScript用のnpm runtime packageは将来追加でき
ますが、v0.1ではpackage間調整を早期に増やさないよう、self-contained clientまたは
小さく安定したruntimeを優先します。

import方向はテストまたはレビューで強制します。`metadata/types/protocol`をleafとし、
contract captureとdispatchはそれらに依存し、HTTPはdispatchに、adapterはHTTPに、
generatorはmetadata/IRに依存します。逆方向にはimportしません。

## 4. Public API候補の比較

構文は重要なpublic decisionです。以下はtrade-offを示すための例であり、承認済みAPI
ではありません。

### 案A: 通常procへのannotation

```nim
type User = object
  id: int
  name: string

proc getUser(id: int): User {.rpc.} =
  # 通常の実装
  ...

# globalなcompile-time stateによる自動探索を避け、明示的に組み立てる
let usersApi = rpcRouter(getUser)
```

signatureとimplementationを分けてannotationする形や、procのcaptureとbindingを同時に
行うregistration macroも検証対象です。

### 案B: Contract block

```nim
contract Users:
  proc get(id: int): User

proc get(id: int): User =
  ...

let usersApi = bind(Users, get)
```

block内にimplementationも書く形も考えられますが、DSLがより多くのNim構文を所有し、
contractとserverの分離が曖昧になる可能性があります。

### 比較

| 観点 | 案A: `{.rpc.}` proc | 案B: `contract` block |
|---|---|---|
| Nimとしての自然さ | 通常の`proc`を装飾するだけで、通常の呼び出し方法を保ち、段階的に導入できるため最も自然です。 | 「contract first」を視覚的に表現できますが、project固有のcommand/macro文法を覚える必要があります。 |
| macro実装の難易度 | 各procのpragma macroは比較的局所的です。一方、宣言と定義、generic、overload、symbol identity、隠れたglobal stateを使わないoperation収集は難所です。 | 1つのmacroがservice全体を見るため、重複名やservice共通metadataを扱いやすくなります。一方、入れ子ASTのparse/validation、symbol生成、implementation bindingの設計が必要です。 |
| 型情報の取得しやすさ | typed proc signatureを直接扱える可能性がありますが、pragma macroの実行phaseとsymbol/type APIをPoCで確認する必要があります。明示的registrationならsymbol identityを保持しやすくなります。 | signatureをまとめて確認できますが、untyped macro内の宣言は当初syntax treeです。alias/genericの解決やimplementationとのbindingに追加のsemantic passが必要になる可能性があります。 |
| エラーメッセージ | compiler errorを通常proc付近に表示でき、未対応parameterも局所的に指摘できます。generated companion symbol由来のerrorには対策が必要です。 | operation間errorをまとめて検出できますが、DSL構文errorや生成宣言のerrorは、工夫しないとmacro側の位置を指す可能性があります。 |
| IDE/toolingとの相性 | 通常procなのでcompletion、rename、go-to-definition、documentation、formatterと相性が良いです。生成されたcompanionは見えにくい場合があります。 | 基本的なproc構文はhighlightできますが、DSLが生成するname/symbolやcontractからimplementationへのnavigationは不透明になり得ます。 |
| 将来拡張性 | operation単位pragmaでroute、auth metadata、deprecation、declared errorを追加できます。一方、optionが増えるとannotation過多になり、group/versionには明示的なservice/router概念が必要です。 | service name、共通path/version、middleware metadata、operation間constraintを自然に所有できます。一方、大きなclosed DSLへ成長し、通常のNim機能を扱いにくくする危険があります。 |

### 最初のPoCへの提案

最初は**案A（通常proc + 小さなpragma + 明示的registration）**を試します。Nimらしく、
toolingとの相性がよいbaselineであり、新しい構文を最小化できます。program内の全
`{.rpc.}`を自動探索せず、contractを決定的かつmodularにする明示的なassembly pointを
要求します。同時に、案Bについては同じcontract fixtureを用いたcompile-time spike
だけを作り、ASTの品質とdiagnosticを比較します。

両案の長所を組み合わせる場合も、block DSLをすぐ確定する必要はありません。通常の
annotated procを`rpcService("Users", getUser, ...)`のような明示的宣言でまとめる形が
候補です。declarationとimplementationを分けるべきかもPoCの結果で判断します。名称を
exportする前にmaintainerレビューを必須とします。

## 5. 最初のPoCで検証するNim固有の課題

未確認のmacro上の仮定をframeworkへ組み込まず、捨てfixtureまたはcompile-only testで
検証します。

1. **Pragma macro semantics:** Nim 2.2.xで、forward declaration、body付きproc、
   exported proc、async proc、method、generic proc、overload、default argument、
   `var`/`ref`/`sink` parameter、calling convention/effect pragmaについて、取得できる
   ASTおよびsemantic type情報を確認します。
2. **安定したtype introspection:** alias、distinct type、enum、inheritance/variantを
   持つobject、tuple、`Option`、sequence、array、table、ref、recursive type、別moduleの
   型について`getTypeInst`、`getTypeImpl`、symbol implementationを比較します。
   compiler crashを起こさずcycleを検出できることも確認します。
3. **Macro hygieneとmodule境界:** generated nameが衝突せず、source locationが有用で、
   private/exported typeを予測可能に扱え、global cacheなしで別moduleからcontractを
   import/registerできることを確認します。
4. **Metadata materialization:** canonical modelをtyped constant、generated code、
   deterministic artifactのどれとして出力するか検証します。machineやincremental build
   によらず順序が再現可能であることも確認します。
5. **Handler erasure:** 異なるproc signatureを、unsafe cast、過大なgenerated branch、
   async behaviorの消失なしに1つのdispatcherへbindできるか検証します。untyped JSON
   境界の周囲にtyped adapterを生成する方式を優先します。
6. **JSON behavior:** missingとnull、integer range、enum表現、unknown object field、
   variant object、custom serializer、recursion、diagnostic pathについて、採用候補のNim
   JSON libraryの挙動を確認し、v0.1の対応subsetを明記します。
7. **Nim typeからTypeScriptへのmapping:** identifier escaping、reserved word、optionalと
   nullable field、integer precision、recursive declaration、enum、generic、
   discriminated union、決定的なdependency orderingを確認します。
8. **Compile-time diagnostic:** negative fixtureを作り、errorがcompiler stack traceや
   generated identifierではなく、operation/parameter path付きでuser declarationを
   指すことを確認します。
9. **別processでのgeneration:** application startupを実行せず、CLIがcontract metadataを
   loadまたは受信する方法と、cross compilationが出力へ影響するかを確認します。
   server buildのたびにTypeScript生成を強制しないようにします。
10. **Optional adapter dependency:** Mummyなしでcoreをcompile/testでき、Nimble packagingで
    adapter dependencyを分離できることを確認します。その後、Mummy adapterのrequest
    lifetime、async model、route parameter、response ownershipを確認します。
11. **Nim target:** serverはnative Nimを優先しますが、将来のNim clientを誤って阻害
    しないよう、どのcore typeがNimのJavaScript targetへcompileできるか調べます。
    これは調査項目であり、v0.1の約束ではありません。

## 6. 初期方針とレビュー必須事項

framework非依存core、明示的registration、unary HTTP/JSON、決定的code generation、
意図的に限定したtype setは安全な初期方針です。一方、public macroの名称と構文、
metadataの安定性、route/envelope format、serialization rule、typed error宣言構文、
async abstraction、対応type matrix、最初のadapterを同じNimble packageで配布するかは、
レビュー必須事項として残します。
