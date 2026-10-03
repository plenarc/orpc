# 小さな実装ロードマップ

**状態:** 提案中。設計レビュー完了後にのみ実装を開始する

各stepを独立してレビュー可能にします。前のmilestoneの判断とテストが承認されてから
次へ進みます。

## Phase 0 — 判断材料を得るためのspike

1. Nim 2.2.xのCI/toolchain matrixを固定し、承認後にのみ最小Nimble packageを追加する。
2. pragma-proc案とcontract-block案のcompile-only fixtureを作る。
3. 両案のAST、取得可能な型情報、diagnostic、module境界での挙動を記録する。
4. 決定的なmetadata exportとTypeScript生成を個別に試す。
5. v0.1の宣言形式、対応type subset、metadata ownershipを選ぶ短いdecision recordを
   公開し、maintainerの承認を得る。

**完了条件:** server frameworkはまだ追加しない。riskの高いmacro上の疑問に根拠を
持って回答でき、public APIが承認されている。

## Phase 1 — 正規化contract model

1. 最小限のoperation/type/error metadata structureを実装する。
2. 承認済み宣言構文のcaptureを実装する。
3. 未対応型をsource位置の分かるcompile-time errorで拒否する。
4. positive/negative compile fixtureと決定的metadata snapshotを追加する。

**完了条件:** HTTPやMummyへ依存せず、1つのcontractから安定したmetadataを生成できる。

## Phase 2 — TypeScript generator

1. 小さなgenerator向けIRと安定したnaming/ordering ruleを定義する。
2. 承認済みNim type matrixからTypeScript request/result型を生成する。
3. 注入可能な`fetch`互換transportを使うframework非依存clientを生成する。
4. 固定したTypeScript versionでgolden outputをtype-checkし、escaping、optionality、
   nullability、recursion policy、error decodingをテストする。

**完了条件:** generated codeがcompileでき、React/Vue/Svelte/Solidに依存しない。

## Phase 3 — DispatchとHTTP/JSON

1. 明示的registrationを通じ、1つのsynchronous handlerをbindする。
2. 承認済みtype subsetのstrictなrequest decodeとresponse encodeを追加する。
3. protocol error、status mapping、content type、size limit、想定外exceptionのsanitizeを
   定義してテストする。
4. generated clientのrequest shapeからhandler resultまでin-memory round tripする。

**完了条件:** coreのvertical sliceがframework非依存HTTP abstraction上で動作する。

## Phase 4 — 最初のadapter

1. MummyのAPI、lifecycle、async constraintをadapter seamに照らして検証する。
2. oRPC coreへ内向きに依存するoptional module/packageとしてMummy adapterを実装する。
3. 実際のHTTP round-trip integration testと最小exampleを追加する。
4. Mummyをinstall/importせずcoreをbuild/testできることを確認する。

**完了条件:** browserで利用可能なTypeScript clientからHTTP経由でNim handlerを呼べ、
Web adapterを置き換えてもcontractが変わらない。

## Phase 5 — v0.1 hardening

1. v0.1のwire behaviorと対応type matrixだけをfreezeし、文書化する。
2. 対応OS/toolchainのCI、release packaging check、deterministic generation checkを追加する。
3. compatibility fixture、security limit、structured diagnostic、changelog/versioning policyを
   追加する。
4. compile time、生成size、request overhead、concurrency behaviorを測定し、未対応workloadを
   約束せず結果を公開する。
5. release candidateでproduction-orientedなfeedbackを集めてからv0.1をtagする。

## v0.1以降の候補（個別提案が必要）

- 正規化modelからのOpenAPI 3.1生成
- 最小error vocabularyを超えるdeclared typed-error payload
- Nim client生成
- 追加framework adapter
- compatibility/diff tooling、streaming、alternate transport、より豊富なvalidator

これらは最初のvertical sliceを検証する前提条件にはしません。Nim frontend/reactivity
systemは別の長期的project concernであり、このroadmapの対象外です。
