---
title: "React + Rails 技術スタック"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rails", "React"]
published: false
---

こんにちは！machampです。

この記事では、React を採用した理由、それによって生じた技術的な課題、その課題に対してどう対応したか、
またどのように開発を進めていったか紹介しようと思います。

## React を採用した理由

今回私が担当した案件では以下のような課題がありました。

- 複雑なフォームの実装。(入力項目が多い、配列要素の操作、配列の中にさらに配列がある)
- さらにユーザーからの要望が複雑化する可能性があった
- 利用制限が厳しい外部サービスとの連携
- モーダル上で操作することが多い

私は今まで `jQuery` や `Stimulus` を使用した開発しか経験がありませんでした。

- 最初はどのようなフォルダ構造にすれば良いのか分からなかった
    - 初期状態でRailsの様に `controllers`  `models` のようなフォルダが存在しないから
        - 検索していると参考になりそうなリポジトリが見つかったからそれを参考にした
            - `features` `pages` の様なフォルダを作成していった
    - `pages` `components` `features` `lib` のような構造になっている
        - pages
            - 最終的なアウトプット。 `components` `features` を呼び出してページを構成。
            - Rails の `views` に相当するフォルダ
        - features
            - 顧客のご要望を実現するための機能をまとめたフォルダ
            - API を呼び出すコードやフォームを含む
        - components
            - `Button`  や `Modal` のような汎用的なコンポーネントを置くフォルダ
        - lib
            - アプリケーション全体で使用されるコード
            - `axios` や `react-query` などを含む
    - `features` の構成について
        - components
            - `CreateReservation.tsx` や `ListReservations` で構成されている
        - hooks
            - useCreateReservation
        - schemas
    - フォルダの構造が上記の通りで依存についても注意した
        - ここはまだ書かない
        - 上位のフォルダから下位のフォルダに依存するようにした
        - `pages` は `components` `features` に依存し、 `features` は `components` に依存するといったように
        - 実際、このような設計にすることである程度アプリケーションが出来てくると、変更がある場合は、新しい feature を追加するか既存の feature に変更が入るか
- 状態管理について
    - React の状態管理は redux, Recoil, jotai などたくさんライブラリが存在してどれを採用するのが良いのか分からない
    - なんとなく学習コストが高いイメージがあった
    - まず、今回の要件でどのような「状態」を管理する必要があるのかを考えた。記事を参考にした。
        - Server Cache State
            - Tanstack Query
        - Global State
            - context + hooks
            - [React-toastify](https://fkhadra.github.io/react-toastify/introduction)
    - 上記に加えてフォームの状態も管理する必要がある
        - これは react-hook-form を採用した
    - なんとなく React といったら Redux！というイメージがありましたが、今は要件によっては状態管理のライブラリは採用しなくても良いのかなーと思いました。
- **Storybook**
    - コンポーネントのカタログとしてではなく、UI の開発を効率よく進めるために Storybook も導入しました
    - 導入目的
        - [Args](https://storybook.js.org/docs/writing-stories/args) で UI の振る舞いをテストする
        - Play function でインタラクションをテストする
            - サンプルコードを書く
    - API から取得したデータを表示する部分は MSW を使用した
