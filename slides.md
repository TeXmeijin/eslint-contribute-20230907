---
class: text-center
highlighter: shiki
colorSchema: light
lineNumbers: false
drawings:
  persist: false
transition: slide-left
title: 初めてESLintプラグインにコントリビュートした話
---

# 初めてESLintプラグインに<br>コントリビュートした話

## 〜ESLintルール作成のすゝめ〜

@meijin_garden / 株式会社NoSchool CTO

---

<style scoped>
h2 {
  margin-top: 64px;
  margin-bottom: 16px;
}
</style>

# お話する内容

1. ESLintのルール設定にこだわる意義
2. 関わっているプロダクトで見つけた課題
3. OSSにコントリビュートした内容

## 対象読者

- ESLintを使ったことはあるけど、使う意義があまりわかっていない方
- ESLintのルールを自前で実装するイメージが湧いていない方
- OSSコントリビュートのハードルが高いと思っている方

---

# 自己紹介

- 名人
  - Twitter(X): [名人｜マナリンクCTO](https://twitter.com/meijin_garden)
  - Zenn: https://zenn.dev/texmeijin
- 株式会社NoSchool CTO
  - オンライン家庭教師マナリンク(https://manalink.jp/)
- 個人開発
  - テストメーカー(https://test-maker.app/)
- 好きな言語はTypeScript、好きなHTTPヘッダーはContent-Disposition
- 趣味
  - 将棋☗、カメラ📸、ラム酒🥃、個人開発💻、筋トレ💪、高校野球観戦⚾

<div class="absolute top-12 right-12">
<img class="w-60 h-60 rounded-full" src="https://github.com/TeXmeijin/vite-react-ts-tailwind-firebase-starter/assets/7464929/09bd0b32-0bcc-4f0d-849a-ccfdd46713ba" alt="">
</div>

---
layout: cover
---

## ESLintのルール設定にこだわると嬉しいこと

---

# 簡単な例え話

〜あるところに、うっかりデバッグ用のconsole.logを含んで提出されたPull Requestに怒る人がいました〜

<img class="h-[400px] mx-auto" src="https://github.com/TeXmeijin/vite-react-ts-tailwind-firebase-starter/assets/7464929/699a4ec1-a562-47ef-9a78-a889392270d6" alt="">

---

# 課題を「人」の問題と「仕組み」の問題に切り分ける

<img class="h-[400px] mx-auto" src="https://github.com/TeXmeijin/vite-react-ts-tailwind-firebase-starter/assets/7464929/148002e1-72b4-425c-a6db-f5104a4c889a" alt="">

---

# 仕組みで防ぐ

- git commit時
  - lint-staged/husky で変更ファイルに対してLintを実行する
  - またはIDEでできるように設定ファイルを配布するなどもあり
- Pull Request時
  - CIでLintを実行する
  - reviewdogやeslint-changed-filesを使えば、変更ファイルのみを対象にLintを実行できる

---

# ESLintでできること

- console.logの入れっぱなしなどイージーミスを防ぐ
  - https://eslint.org/docs/latest/rules/no-console
- いわゆる書き方の好みの問題をPrj内で統一する
  - https://github.com/jsx-eslint/eslint-plugin-react/blob/master/docs/rules/jsx-boolean-value.md
- 見やすくするが、手作業するかしないかが人によって分かれるやつ
  - https://github.com/import-js/eslint-plugin-import/blob/main/docs/rules/order.md
- 社内で決めたアーキテクチャを統一する
  - https://www.npmjs.com/package/eslint-plugin-strict-dependencies
    - **これが本スライドで話す主題です**

---

# コントリビュートしたESLintプラグイン

eslint-plugin-strict-dependencies

![screenshot](https://github.com/TeXmeijin/vite-react-ts-tailwind-firebase-starter/assets/7464929/b06ed8ac-2139-4a7c-8d95-6265f2b5455f)

---

# どんなプラグイン（だった）か

- あるモジュールから別のモジュールをimportできる・できないのルールを規定する
  - `.eslintrc.js`に設定を書き、破られていたらエラー扱いとする
  - husky/lint-stagedやCIで強制できる
- 利用例1：プロダクトで決めたアーキテクチャの徹底
  - `src/components/page`は`src/pages`からしか呼べない
  - `src/components/features`は`src/pages`からしか呼べない
  - `src/components/ui`は`src/components/page`、`src/components/features`からしか呼べない
- 利用例2：外部ライブラリに対する腐敗防止層利用の徹底
  - MUIのコンポーネントは`src/components/ui`からしか呼べない
  - `@sentry/react`は`src/libs/sentry.ts`からしか呼べない
  - `react-icons`は`src/components/ui/icons`からしか呼べない

---

# 設定例（旧eslintrc形式）

```js
module.exports = {
  plugins: ['strict-dependencies'],
  rules: {
    'strict-dependencies/strict-dependencies': [
      'error',
      [
        {
          "module": "src/components/ui",
          "allowReferenceFrom": ["src/components/page"],
          "allowSameModule": true
        },
        {
          "module": "next/router",
          "allowReferenceFrom": ["src/libs/router.ts"],
          "allowSameModule": false
        },
      ]
    ],
  },
}
```

---

# 利用シーンと、そこで見つけた課題

## 背景
- 弊チームでもさっそくアーキテクチャの徹底と、外部ライブラリの利用制限で用いた
- あるとき、メンバーが`react.Suspense`のラッパーを作ってくれた
- なので、`Suspense`を直接呼ぶのではなく作ったラッパーを使うように徹底したい

## 気がついたこと
- 前述の通り`import A from B`でいうところのBを指定するため、「`react`の中の`Suspense`のみ利用範囲を制限したい」ケースには対応できない
- これまで通り設定すると、`react`からのimportが全部NGになってしまう

```js
        {
          "module": "react",
          "allowReferenceFrom": ["src/libs/suspense.ts"],
          "allowSameModule": false
        },
```

---

# どうするか

---

# `import A from B`に対して「**BからAを**importしているとき」というより細かな条件を指定できるように機能追加したい！

イメージ

```js
        {
          "module": "react",
          "targetMembers": ["Suspense"],
          "allowReferenceFrom": ["src/libs/suspense.ts"],
          "allowSameModule": false
        },
```

---

# なんか既存の実装を理解したら、実装できそう
既存の実装を理解したら、

**【import文において、import対象のモジュール名を取得する方法と、対象ファイル名を取得する方法】**

がわかるはずなので、もう少し応用してimportするメンバー名を取得する方法を考えればよさそう。

---

# 機能追加するときは（一旦）ここだけ見る

```js
module.exports = {
  meta: {
    // meta情報なので機能理解にあたってはスルー
  },
  create: (context) => {
    // ここに色々書いてあるのも一旦スルー

    // ここでreturnされたものがプラグインの動作を決めるのでまずはここで全体理解
    return {
      ImportDeclaration: checkImport,
    }
  },
}
```

---

# ざっくり解説

```js
    return {
      ImportDeclaration: checkImport,
    }
```

## `ImportDeclaration`というのは
- import文のこと
- 例：`import A from B`

## `ImportDeclaration: checkImport`と指定することで
- ESLintプログラムがimport文を見つけたら、`checkImport`関数を実行するようになる
- 個人的には脳内で「`onImportDeclarationAppeared: checkImport`」といった風に読み替えて読んでいて、**イベントハンドラをプラグインを通して登録している**と考えるとしっくりきています

---

# `ImportDeclaration`って予約語なの？

- 予約語です（適当に決めたら動きません）
- 他にも`ExportSpecifier`とか`FunctionDeclaration`とか色々ある
- ESLint実行時には、JavaScriptのソースコードをASTという形式に変換するが、そのときソースコード中の各パーツ（ノードという）に当てられるType
- なので、これらの名前はJavaScriptのASTで規定されている
- ESLintって要は「JavaScript内にこういう種類のノードがあったらこういうルールに沿っているかチェックしてね」の集合なので、作りたいルールに応じてASTのノードのTypeを調べて関数を定義していく感じ

---

# AST(Abstract Syntax Tree)とは

- こちらで規定されている：https://github.com/estree/estree
  - ※JavaScriptに限らずどの言語にもある一般的な概念
- ASTは以下のようなツールで見れる
  - https://astexplorer.net/
  - https://ts-ast-viewer.com/

---

# 覚えておくこと

- 全体的に
  - よほどのことがない限り、ASTについて丸暗記したり徹底理解する必要はない
  - 個人的には「**まあ、プログラムをプログラムが解析したり変換するなら、プログラムはただの文字列なので、プログラムが操作可能な形式に変換しないとダメやんな〜**」くらいに思っておく
- `ImportDeclaration: checkImport`における`checkImport`関数について
  - 前述の通り、import文が見つかったときにそのimport文に対して実行する関数
  - 第1引数にASTでパースされた`ImportDeclaration`型のオブジェクトが渡される
  - `ImportDeclaration`型って何やねん。という話ですが、前述のAST Explorerなどで見てもいいし、[typescript-eslint](https://github.com/typescript-eslint/typescript-eslint/blob/main/packages/ast-spec/src/declaration/ImportDeclaration/spec.ts)に型情報が記載されているのでそちらを見る

---

# 実装方針

- 前述の知識から、今回の目的の一つである「import対象のメンバー名を取得する」方法は`node.specifiers`を使う
- `node.specifiers`にImport対象が入っていて、それぞれこんな形状だが、Default Importは`imported.name`が無い罠がある（import文って3種類あんねん）

```js
const importedModules = node.specifiers.filter(spec => 'imported' in spec).map(spec => spec.imported.name)
```

```ts
import type { ImportDefaultSpecifier } from '../special/ImportDefaultSpecifier/spec';
import type { ImportNamespaceSpecifier } from '../special/ImportNamespaceSpecifier/spec';
import type { ImportSpecifier } from '../special/ImportSpecifier/spec';

export type ImportClause =
  | ImportDefaultSpecifier
  | ImportNamespaceSpecifier
  | ImportSpecifier;
```


---

# テストコードと動作確認

- 本プラグインはありがたいことにテストコードが用意されていたので、手元にCloneして実装した後にデグレがないか実行
- 前述したImportDeclarationが複数パターンある件にハマらないため、モックデータの拡充もついでにした

## ローカルでの動作確認

- 方法は複数あると思うが、`yarn`や`npm`はローカルにCloneしたモジュールをinstallすることもできるので、手元で改修後のプラグインをinstallして自社プロダクトにて動作確認した
  - e.g. `yarn add -D ../../../hoge/eslint-plugin-strict-dependencies`

---

# Pull Request提出〜マージまで

- 2023年6月30日16時頃：弊社メンバーからSuspenseラッパー実装の発案があり、それに伴ってプラグインへの機能追加を思いつく
- 6月30日18時頃：なんとなく動くやつができる
- 7月2日12時頃：テストコードを書き、動作確認もできたのでPRを提出
  - https://github.com/knowledge-work/eslint-plugin-strict-dependencies/pull/12
  - 和製OSSなので日本語で書けたのがありがたい
- 8月18日：なんだかんだあってPull Requestをマージしていただけた🎉

※今回ESLintプラグインへのコントリビュートは初めてでしたが、ESLintプラグインの作り方自体は昨年から知ってはいました。なので機能追加したいときにすぐに動けたと思います。今すぐ解決したいIssueがなくても、ESLintプラグインの作り方をざっくり知っておくといつか使えるかもしれません

---
layout: cover
---

# まとめ

---

# まとめ

- ESLintのルール設定にこだわると嬉しいこと
  - 人の問題と仕組みの問題に切り分けられる
- ESLintプラグインを作る/機能追加するときは
  - ASTの知識は必要だが、丸暗記する必要はない
  - 既存の実装を読んで、どういうノードがあるか、どういうノードを取得すればいいかを理解する
  - ESLintプラグインでどんなことができるのかを知っておくと、いつかタイミングが来たときに役に立つ

---
layout: cover
---

# 宣伝


---

# 「マナリンク」について

- オンライン家庭教師マナリンク(https://manalink.jp/)
- コロナ禍から増え始めた新しい教育の仕事である「オンライン家庭教師」を広めるスタートアップ
- 先生と保護者様のマッチングサイトと、指導開始後の宿題や指導料金の管理等のツールを提供しています

<div class="mt-4 flex gap-x-4 items-center">
<img class="h-[300px]" src="https://github.com/TeXmeijin/vite-react-ts-tailwind-firebase-starter/assets/7464929/cbabe464-3064-49e0-aba8-5c440b7c1eb1" alt="">
<img class="h-[260px]" src="https://github.com/TeXmeijin/vite-react-ts-tailwind-firebase-starter/assets/7464929/9a8e53a2-0257-44a5-9d8d-51a24eac0b1f" alt="">
</div>

---

# 弊社の開発チームについて

- メンバー構成
  - 全4名（CTO、フルスタック2名、React Nativeエンジニア1名）
- 【仕組みを憎んで人を憎まず】
  - 毎月**最大3営業日程度**「仕組み化・自動化」に関する工数を使います
  - 実績の一例
    - ローカル環境の色々なデータの自動生成・破棄コマンドの作成
    - PHPStanの導入と設定
    - SQLのSlow Query検知やN+1の自動テスト時の検知
    - Mock Service Workerの導入とテストコードへの統合
    - renovateによるライブラリバージョンアップの自動化
- 勉強会
  - 1年以上、週1〜2回の社内勉強会を続けています（※業務時間内）
  - https://zenn.dev/manalink_dev/articles/manalink-study-meetup-history-front-and-network

---

# 募集内容

- 開発メンバーを随時募集しているのですが、いきなり面接等は敷居が高いと思うので
- 以下募集しています！
  - 弊社の社内勉強会にゲスト参加✏️
    - 平日15時〜15時半頃
    - 平日夜
  - 弊社メンバーとレンタルジムを借りて合同筋トレ💪
  - 弊社メンバーと秋葉原の国内最大級のボルダリング上で壁登り🧱
  - 普通にカジュアル面談（オンライン30min）
  - バーにお酒🥃を飲みに行く（私はラム酒がおすすめなのでラム酒デビューしたい方布教させて）

---
layout: cover
---

# ご清聴ありがとうございました

この後の懇親会でぜひお話しましょう！
