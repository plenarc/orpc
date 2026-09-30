# oRPC コントリビューター向けルール

- `docs/project-brief.md`をプロジェクト方針およびスコープの基準とする。
- coreはWeb framework非依存に保つ。core moduleからMummy、Prologue、HappyX、
  frontend frameworkをimportしてはならない。連携コードはadapter packageに置く。
- 暗黙的なglobal registrationや大規模DSLより、小さく明示的でNimらしいAPIを
  優先する。public APIの追加・変更は、実装前に人間のレビューを受ける。
- architecture、wire protocol、互換性に関する大きな判断を暗黙に行わない。
  選択肢とtrade-offを`docs/`に記録し、レビューを依頼する。
- 依存は最小限に保ち、production dependencyを追加する場合は理由を明記する。
- contract metadata、transport非依存dispatch、HTTP/JSON mapping、framework
  adapter、generatorを分離し、それぞれ単独でテスト可能にする。
- レビュー済みの方針変更がない限り、Nim 2.2.xを対象とする。
- compile-time behavior、生成結果、protocol behavior、failure caseに焦点を当てた
  テストを追加する。生成物の決定性と、問題箇所が分かるdiagnosticを重視する。
- oRPC core内でNim frontendまたはreactivity frameworkの開発を開始しない。
- ドキュメントおよびexampleを実装と同期する。提案は提案であると明示し、未実装の
  機能を利用可能であるかのように記載しない。
- file headerが適切な新規project-owned source fileにはMIT Licenseを適用する。
  secret、build cache、local生成物をcommitしない。
