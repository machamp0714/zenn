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

`pages` フォルダには `#root` DOM 配下に追加される最上位のコンポーネントを配置しました。
ページコンポーネントは `features` と `components` のファイルで構成され、各ページコンポーネントは必ずStorybook のストーリーを持ちます。

TODO: ここで Storybook の解説をしても良さそう

##### features

| フォルダ | 役割 |
| --- | --- |
| components | 機能固有のコンポーネント 例) CreateTodo.tsx, DeleteTodo.tsx… |
| api | TanStack Query を使用した API フック |
| schemas| フォームで使用する zod スキーマ定義 |
| hooks | 機能固有のカスタム hooks |
| types | OpenAPI で自動生成された TypeScript の型 |

bulletproof-react の構成から変更した主なポイントは、フォームスキーマを api から分離し、schemas として独立させた所です。今回フォームの実装には [react-hook-form](https://react-hook-form.com/) と [zod](https://zod.dev/) を採用したのですが、zod で定義された下記のようなコードを配置しています。

```typescript
const todoSchema = z.object({
  title: z.string(),
  status: z.enum(['completed', 'incomplete'])
});

type TodoSchema = z.infer<typeof todoSchema>;
```

##### types

`types` フォルダには OpenAPI のスキーマから生成された型を配置しました。
OpenAPI に関しては後述します。

```typescript
import type { components } from '@/swagger';

export type Todo = components['schemas']['todo'];
```

### 状態管理

React の状態管理には Redux, Jotai, Zustand...など多くの選択肢があり、どれを採用すれば良いのかかなり悩みました。そんな中、[「3種類」で管理するReactのState戦略](https://zenn.dev/knowledgework/articles/607ec0c9b0408d) という記事と出会ったのですが、React で扱う状態は以下の3種類に分類できる、という考え方に影響を受けました。

1. Local State（コンポーネント内の状態）
2. Global State（アプリケーション全体で共有する状態）
3. Server State（サーバーから取得するデータの状態）

この分類に基づいて、プロジェクトに最適な状態管理の方法を以下のように決定しました。

#### 1. Local State

コンポーネント単位の状態管理には、React の `useState` を使用しています。



- モーダルの open / close

#### 2. Global State

ページをまたいで保持し続ける必要のある状態のことで、本件では以下の状態を管理する必要がありました。

- 認証情報
- ページをまたぐトースト通知

最初は Jotai などのライブラリの採用も検討しましたが、以下の理由から React の [Context API](https://ja.react.dev/learn/passing-data-deeply-with-context) を採用しました。

- 管理が必要な状態が限定的
- `Context API` で十分カバーできる規模感
- 学習コストの最小化
- [React-Toastify](https://fkhadra.github.io/react-toastify/introduction/) のようなライブラリの活用

学習コストも抑えることができて良い選択だったと思います。また、状態管理ライブラリの選定に関してはこちらの記事も参考になるかと思います。
https://tech.buysell-technologies.com/entry/2024/10/31/100000#Context-API%E3%81%AE%E6%80%9D%E6%83%B3%E3%81%A8%E3%81%AE%E3%82%BA%E3%83%AC

#### 3. Server State

サーバーデータの管理にはデータ以外にも
- データのキャッシュ
- ローディング状態
- 成功/失敗のステータス

といった非同期の操作に関連して変化する状態も管理する必要があります。
これらを効率的に管理するため、bulletproof-react でも採用されている Tanstack Query を選択しました。
実装例は以下のようにシンプルです。

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
Tanstack Query 公式からもリンクされている [TkDodo's blog](https://tkdodo.eu/blog/practical-react-query) に主な使い方がまとめられているので、こちらを読んでおけば実装面でそこまで苦労することはないと思います。特に [Inside React Query](https://tkdodo.eu/blog/inside-react-query) は内部で何が起きているのかをざっくり解説してくれているので、一読をお勧めします。


### 開発の進め方

フォルダの構造も決まって、使用するライブラリも程度決めることができました。そこで次に「どうやって UI を効率よく開発していくか」ということを考えることにしました。

今回のプロジェクトで目標としたことは下記の３点です。

1. APIの開発を待たずにフロントエンド開発を進められる体制作り
2. バックエンドから受け取る多様なデータ状態の網羅的なテスト
3. ユーザー権限に応じたUI変更（操作可否など）の効率的なテスト

これらの課題に対して、Storybook と [MSW（Mock Service Worker）](https://mswjs.io/)を組み合わせたアプローチを採用しました。

#### 1. APIモックによる開発の並行化

MSW は API リクエストをインターセプトすることで、リクエストに対してモックされたレスポンスで応答することが出来るので、実際のAPIの完成を待たずに開発を進めることができます。具体的には下記のようなハンドラーを実装していくことになります。

```typescript
import { http, HttpResponse } from 'msw'

export const todoHandlers = () => [
  http.get<never, never, TodosResponse, '/todos'>('/todos', () => {
    return HttpResponse.json(todos)
  }),
  http.post<TodoParams, never, Todo, '/todo/:id'>('/todo/:id', ({ params }) => {
    const { id } = params;
    const todo = todos.find(todo => todo.id === id);

    if (todo) {
      return HttpResponse.json(todo)
    }

    return new HttpResponse(null, { status: 404 })
  })
];
```

Storybook で MSW を使用するには [msw-storybook-addon](https://storybook.js.org/addons/msw-storybook-addon) というアドオンを追加する必要があります。

#### 2. 多様なデータ状態のテスト

各状態のStoryを定義することで、様々なデータ状態を一覧で確認できるようにしました。
新しく入ったメンバーが一々自分でデータの準備をすることなく、UI を閲覧できるように
しておきたかったというのもあります。

```ts
export default {
  title: 'Pages/Todo',
  component: Todo,
  parameters: {
    msw: {
      handlers: todoHandlers,
    },
  },
} as Meta<typeof Todo>;

export const Completed = {
  name: '完了'
}

export const Incomplete = {
  name: '未完了'
}
```

#### 3. ユーザー権限に応じたUI変更のテスト

Storybook の ArgTypes と Decorators を組み合わせて、ユーザー権限の動的な切り替えを実現しました。

```tsx
export default {
  title: 'Pages/Welcome',
  component: Welcome,
  argTypes: {
    role: {
      type: { name: 'string', required: true },
      control: { type: 'radio' },
      options: ['管理者', '一般'],
    },
  }
} as Meta<Args>;
```
このように argTypes を追加して Storybook の Control タブでログインユーザーの権限を切り替えられるようにします。ただし、ログインユーザーの情報は Context API で管理しているため、これだけでは権限は変わりません。

Storybook は Decorators を使用して Story をラップすることが出来るので、 ArgTypes で選択した値に応じたユーザーのオブジェクトを Context に渡す Decorator を自作しました。

```tsx
const currentUserDecorator = (Story, context) => {
  const role = context.args.role; // ArgTypesで選択した値はcontextから取得できる
  let currentUser;
	
  if (role === '一般') {
    currentUser = generalUser;
  } else {
    currentUser = adminUser;
  }
	
  return (
    <CurrentUserContext.Provider value={currentUser}>
      <Story />
    </CurrentUserContext.Provider>
  )
}
```

## 4. APIレスポンスの型を自動生成する

TypeScript を採用する大きなメリットの一つは、型安全性です。
せっかく TypeScript を使用しているので API のレスポンスの型が欲しいです。

### 当初の構想：zod による手動定義

最初は zod を使用して型を手動で定義し、ランタイムでの型検証を行うことを検討しました。

```ts
const TodoResponseSchema = z.object({
  id: z.number().required(),
  title: z.string().required(),
  status: z.enum(['completed', 'incompleted'])
});

type TodoResponse = z.infer<typeof TodoResponseSchema>;

const getTodos = async () => {
  const response = await axios.get<TodoResponse>('/api/v1/todos');
  
  return TodoResponseSchema.parse(response.data);
}
```
しかし、この手法には以下の課題があったのでボツにしました。

- API の仕様変更時に手動での型定義の更新が必要
- 必須項目の変更時に対応漏れが発生するリスク
- 型定義とAPI仕様の乖離が生じる可能性

### OpenAPI の採用

上記の課題を解決するため、OpenAPI を採用して型を自動生成する方針に切り替えました。

:::message
**OpenAPI とは**

OpenAPI は RESTful API の仕様を記述するための標準フォーマットです。
- YAML または JSON 形式で記述可能
- Swagger UI などのドキュメント生成ツールと連携可能
- 型定義の自動生成に活用可能
:::

例えば、Todo API の仕様は以下のように記述することができます。

```yaml
openapi: 3.0.1
info:
  title: Sample API
  version: 0.1.0

paths:
  /todos:
    get:
      summary: Todo一覧
      responses:
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/todo'
components:
  schemas:
    todo:
      type: object
      properties:
        id:
          type: integer
        title:
          type: string
        status:
          type: string
          enum:
            - completed
            - incomplete
```

### Stoplight による仕様定義

YAML で書く場合は上記のように記述するのですが、これを手で書いていくのは大変なので、私は
Stoplight という GUI 上で API の仕様を定義できるツールを使用しています。

https://stoplight.io/

最近知ったのですが、 Apidog というツールもあるみたいです。
Apidog は無料プランでも最大4名のメンバーが利用出来るのでお試しで使いやすかもしれません。

https://apidog.com/jp/

### TypeScript の型生成

さて OpenAPI からどうやって TypeScript の型を生成するかですが、 [openapi-typescript](https://github.com/openapi-ts/openapi-typescript) を利用しました。使い方は簡単で↓を実行するだけです。

```
$ yarn openapi-typescript ./path/to/my/stoplight.yaml -o ./path/to/my/schema.d.ts
```
すると、このような型定義が生成されます。
```ts
export interface paths {
  // 省略...
}

export interface components {
  schemas: {
    todo: {
      id: number,
      title: string,
      /** @enum {string} */
      status: 'completed' | 'incomplete'
    }
  }
}
```

これにより、手動での型定義が不要になり、型定義と API 仕様が一致することが保証された状態でフロントエンドの開発を進められるようになりました。


## 5. 全体の開発フロー

プロジェクト全体の開発フローとして、スキーマ駆動開発を採用しました。

### スキーマ駆動開発とは

スキーマ駆動開発はOpenAPIでAPI仕様を事前に定義し、その仕様に基づいてフロントエンドとバックエンドの開発を進めていく手法です。

フロントエンドでは定義されたレスポンススキーマが返ってくることを想定して開発を進め、
バックエンドでは仕様どおりのレスポンスを返すAPIを実装していきます。

フロントエンドの開発方針は前章までに解説したとおりですので、ここではRailsでOpenAPIを使用した場合のAPI開発について説明します。

### Rails での OpenAPI 実装

Railsでは [rswag](https://github.com/rswag/rswag?tab=readme-ov-file)  という gem を活用して OpenAPI の仕様を生成・管理します。rswag は request spec を OpenAPI ベースの DSL で拡張し、テストと API ドキュメントの生成を同時に行えます。

:::details rswag のサンプルコード
```rb
# spec/requests/todos_spec.rb

require 'swagger_helper'

describe 'Todos API' do
  path '/todos' do
    get 'Todo一覧' do
      tags 'Todo'
      produces 'application/json'
			
      response '200', 'OK' do
        schema type: :array,
               items: { '$ref' => '#/components/schemas/todo' }

        run_test!
      end
    end
  end
end

# spec/swagger_helper.rb

config.openapi_specs = {
  schemas: {
    todo: {
      type: :object,
      properties: {
        id: { type: :integer },
        title: { type: :string },
        status: {
          type: :string,
          enum: %w[completed incomplete]
        }
      }
    }
  }
}
```
:::

OpenAPI の YAML フォーマットと似た記法で request spec を書くことができ、テストがパスすることを確認した後、`rails rswag:specs:swaggerize` を実行して YAML を生成します。
生成された YAML は rswag-ui を使用して公開でき、Stoplight で生成した YAML と共に管理できます。

```rb
Rswag::Ui.configure do |c|
  c.openapi_endpoint '/api-docs/v1/swagger.yaml', 'API V1 Docs'
  c.openapi_endpoint '/api-docs/v1/stoplight.yaml', 'API V1 Stoplight Docs'
end
```

### スキーマ駆動開発のメリット・デメリット

#### メリット

##### 実装に着手する前に API の仕様をレビューできる

APIインターフェースの設計について早期にレビューすることで、データ構造やリクエストの問題点を事前に発見できます。

##### フロントエンドとバックエンドを並行して開発できる

スキーマを定義した時点でフロントエンドとバックエンドの開発を同時にスタートできるため、開発の待ち時間を最小限に抑えられます。
これはプロジェクト管理の面でも効果的で、タスクの並行作業が可能になり、作業の振り分けもスムーズになりました。

##### 仕様書と実装の隔離が発生しにくくなる

スキーマを単一の情報源（Single Source of Truth）として扱うことで、API 仕様書が常に最新の状態を保てます。実装とドキュメントが別々に管理される場合によくある仕様と実装の不一致という問題を防げます。

#### デメリット

##### 技術的な学習コスト

Stoplight のようなツールを使用して YAML を作成するとしても、OpenAPI の記法の学習は
必須なので学習コストは発生します。（ツールの使い方自体はすぐ慣れると思います）
また、rswag を使うことで request spec の書き方が従来の書き方とは異なるのでそこも慣れが必要になります。

ただし、これらの学習コストを考慮しても、レスポンスが仕様通りであることが担保されているという安心感があるのは価値があると思います。単一の情報源は開発の良き友になってくれると思います。
