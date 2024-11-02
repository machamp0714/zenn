---
title: "jQueryしか使ったことがないRailsエンジニアがReactを採用したお話"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rails", "React"]
published: false
---

## はじめに

この記事は、jQuery で Rails アプリケーションのフロントエンド開発をしてきた私が、
初めて React を採用したプロジェクトでの経験をお伝えするものです。技術選定から実際の開発
まで、一貫して関わる機会を得て、改めて自分の考えを整理する目的に記事にしました。

具体的には以下のような内容について書いたので、興味があれば最後までお付き合い頂けますと幸いです。
- なぜ React を選んだのか
- 開発中に直面した課題とその解決方法
- Rails × React での開発の進め方

## 1. Reactを採用した理由

技術選定にあたり、まず Rails7 から標準で組み込まれている Hotwire の採用を検討しました。
検討の際は [Hotwireの良かった点、辛かった点、向いているケース、向いていないケース](Hotwireの良かった点、辛かった点、向いているケース、向いていないケース) という記事を参考にしました。

記事中で

> Hotwireを使う際にはStimulus（というかJS）をできるだけ書かずに、サーバーサイドにロジックを寄せてTurboで処理するというのもポイントかなと思います。JSを書く量が増えるほどHotwireの良さが消えていき、React + TSを使いたくなります。

こう書かれていますが、まさにこちらの指摘は今回のプロジェクトの要件に関係していました。
具体的には以下の理由から、Hotwire の採用を見送り、React + TypeScript の採用を検討し始めました。

### フロントエンドのロジック量

Hotwire は「できるだけ JavaScript を書かずに、サーバーサイドにロジックを寄せる」というアプローチが特徴です。
しかし、今回のプロジェクトでは以下の要件があり、フロントエンドでの JavaScript の記述量が必然的に多くなることが予想されました：

- サーバ上のデータの多様な状態に応じて変化する UI の実装
- インタラクティブな UI の実装

### UI の管理のしやすさ

また、以下のような課題もありました。
- ユーザーの権限によって操作できるオブジェクトが変わる
- 各状態ごとの UI の把握が困難

これらに対しては Storybook を採用するのが良いと判断しました。
(Storybook に関しては後でまた解説します)

- データの状態ごとに UI を整理して管理できる
- ユーザーの権限を切り替えて UI のテストができる
- React と相性が良い

### 結論

最終的に、以下の理由から React + TypeScript の採用を決定しました。

1. フロントエンドでのロジック処理に適している
2. 型安全性による開発の安全性向上
3. Storybook との組み合わせによる UI のテストのやりやすさ向上

## 2. Reactを使えるようにする方法

Rails アプリケーションで React を使用する方法を検討する際、[mastodon](https://github.com/mastodon/mastodon) の実装を参考にしました。mastodon では、バックエンド（Rails）とフロントエンド（React）を完全に分離せず、うまく共存させています。

### mastodon の実装

mastodon では React と haml が混在する構成を採用しており、以下のような特徴があります。

1. 基本的な HTML 構造は Rails 側で生成
2. React コンポーネントをマウントする DOM 要素を用意
3. その DOM 要素に対して React コンポーネントを注入

このアプローチは以下のメリットがあります。

- バックエンドとフロントエンドを分離せずに開発可能
- 既存の Rails の資産（ルーティング、認証、認可）をそのまま活用可能
- 必要な箇所だけを React で実装可能

### 実装手順

具体的な実装は非常にシンプルです。以下の手順で React コンポーネントを画面に表示できます

1. まず、Rails のビューで React をマウントする要素を用意します

```erb
<%# app/views/todos/index.html.erb %>
<div id="root"></div>
```

2. 次に、その要素に React コンポーネントをマウントします

```tsx
// app/frontend/src/index.tsx
const mountNode = document.getElementById('root');

if (mountNode) {
  const root = createRoot(mountNode);

  root.render(
    <React.StrictMode>
      <App />
    </React.StrictMode>
  );
}
```
3. Reactコンポーネントを実装する

```tsx
// app/frontend/src/App.tsx
const App = () => {
  return <div>Hello, World!</div>;
};
```

これだけで React コンポーネントが画面に表示されるようになります。
mastodon ではより複雑な実装を行っていますが、基本的な考え方は同じです。

この方法で React の導入自体は完了しましたが、実際の開発を進めていく上では以下のような課題に直面しました。

- フォルダ構造をどうするか
- 状態管理をどうするか
- 開発の進め方をどうするか

これらの課題については、次の章で詳しく説明していきます。

## 3. フロントエンド開発の課題

### フォルダ設計

フロントエンドのフォルダ構成は、開発の効率性と保守性に大きく影響する重要な要素です。
特に Rails と React が共存する環境では、適切なフォルダ設計が重要になってきます。

#### ベースとなる構成

まず、`rails new` で生成される `javascript` フォルダを `frontend` に変更し、その配下に `src` 
というディレクトリを作成して React のコードをまとめることにしました。

フロントエンドの設計パターンを調査する中で、多くの記事から参照されている [bulletproof-react](https://github.com/alan2207/bulletproof-react) というリポジトリに出会いました。
このリポジトリで採用されている「package by feature」という設計手法は、以下のメリットがあります。

- 機能ごとにフォルダを分けることで、コードの所在が把握しやすい
- 1つのフォルダ内のファイル数を適切に保てる
- 関連するコードが近い場所に配置される

#### 採用したフォルダ構成

最終的には bulletproof-react の [Project Structure](https://github.com/alan2207/bulletproof-react/blob/master/docs/project-structure.md) をベースに、下記のような構成になりました。(ほぼパクリです笑)
```
.
└── src
├── App.tsx
├── mocks
├── components   # 共通コンポーネント
├── features     # 機能単位のコード
├── hooks        # 共通フック
├── index.tsx
├── lib          # ライブラリの設定
├── pages        # ページコンポーネント
├── providers    # Contextプロバイダー
├── routes       # ルーティング設定
├── test         # テストユーティリティ
├── types        # 共通型定義
└── utils        # ユーティリティ関数
```
ただし一部要件に応じて変更を加えた部分もあるのでそこだけ追記しようと思います。

##### pages

- `#root` DOM 配下に追加される最上位のコンポーネントを配置します
- 各ページコンポーネントは必ず Storybook のストーリーを持ちます
- `features` と `components` のファイルで構成されます

##### features ディレクトリ


| フォルダ | 役割 |
| --- | --- |
| components | 機能固有のコンポーネント 例) CreateTodo.tsx, DeleteTodo.tsx… |
| api | TanStack Query を使用した API フック |
| schemas| フォームで使用する zod スキーマ定義 |
| hooks | 機能固有のカスタム hooks |
| types | OpenAPI で自動生成された TypeScript の型 |

bulletproof-react の構成から変更した主なポイントは、フォームスキーマを api から分離し、schemas として独立させた所です。
フォームの実装には [react-hook-form](https://react-hook-form.com/) と [zod](https://zod.dev/) を採用しました。


```typescript
const todoSchema = z.object({
  title: z.string(),
  status: z.enum(['completed', 'incomplete'])
});

type TodoSchema = z.infer<typeof todoSchema>;
```

### 状態管理

React の状態管理には Redux, Jotai, Recoil など多くの選択肢があり、どれを採用すべきか悩むポイントとなります。
私自身、以前 Udemy で Redux を学習した際に、単純な API データの取得でもコードの記述量が多くなることに驚いた経験がありました。

そんな中、[「3種類」で管理するReactのState戦略](https://zenn.dev/knowledgework/articles/607ec0c9b0408d)という記事に出会い、状態管理を以下の3種類に分類するアプローチに大きな影響を受けました：

1. Local State（コンポーネント内の状態）
2. Global State（アプリケーション全体で共有する状態）
3. Server State（サーバーから取得するデータの状態）

この分類に基づいて、プロジェクトに最適な状態管理の方法を以下のように決定しました。

#### 1. Local State

**採用技術**: `React.useState`

コンポーネント単位の状態管理には、React 標準の `useState` を採用しました。
シンプルな実装で十分なケースがほとんどで、余計な複雑さを避けることができました。

#### 2. Global State

**採用技術**: 
- `React.useContext`
- React-Toastify（トースト表示用）

以下の状態を管理する必要がありました：
- 認証情報
- ページを跨ぐトースト通知

当初は Recoil や Jotai の採用も検討しましたが、以下の理由から `useContext` と専用ライブラリの組み合わせを選択しました：

- 管理が必要な状態が限定的
- `useContext` で十分カバーできる規模感
- 学習コストの最小化
- React-Toastify のような実績のあるライブラリの活用

この選択により、実装はシンプルに保たれ、開発チームの学習負荷も最小限に抑えることができました。

#### 3. Server State

**採用技術**: Tanstack Query

サーバーデータの管理には以下の要素が含まれます：
- データのキャッシュ
- ローディング状態
- 成功/失敗のステータス管理
- エラーハンドリング

これらを効率的に管理するため、Bulletproof React でも採用されている Tanstack Query を選択しました。
実装例は以下のようにシンプルです：

```tsx
const getTodos = () => {
  return axios.get('/api/v1/todos');
}

const useTodos = () => {
  return useQuery({
    queryKey: ['todos'],
    queryFn: getTodos
  });
}

const { data, isLoading, isSuccess, isError } = useTodos();
```

Tanstack Query の採用により...

直感的な API での実装が可能
キャッシュ管理の自動化
非同期状態の簡潔な管理
豊富なドキュメントとコミュニティサポート（特に TkDodo's blog の Inside React Query は必読）

まとめ
最終的な状態管理の構成：

Local State → React.useState
Global State → React.useContext + 専用ライブラリ
Server State → Tanstack Query

この構成により、必要十分な状態管理を実現しながら、学習コストと実装の複雑さを最小限に抑えることができました。
特に Server State を Tanstack Query で分離したことで、アプリケーション全体の状態管理がシンプルになりました。

### 開発の進め方

## APIレスポンスの型を自動生成する

## 全体の開発フロー
