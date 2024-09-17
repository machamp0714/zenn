---
title: "React + Rails 技術スタック"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rails", "React"]
published: false
---

### はじめに

今まで jQuery や Stimulus でしかフロントエンドを書いたことがなかった Rails エンジニアが
技術選定から開発まで関われる案件に参画する機会があり、その際に React をフロントエンドの開発に採用したので、なぜ採用したのか、開発上の課題、どう開発を進めたのかを自分の考えを再度整理することも兼ねて記事にしました。

### React を採用した理由

Rails7 からは Hotwire がありますが、今回は採用を見送りました。

Hotwire は全く触ったことがなかったため、 [Hotwireの良かった点、辛かった点、向いているケース、向いていないケース](https://nekorails.hatenablog.com/entry/2022/05/16/170434) を読みましたが、今回私が担当した案件にはマッチしていないと考えました。

記事の中で

> Hotwireを使う際にはStimulus（というかJS）をできるだけ書かずに、サーバーサイドにロジックを寄せてTurboで処理するというのもポイントかなと思います。JSを書く量が増えるほどHotwireの良さが消えていき、React + TSを使いたくなります。
> 

という指摘がありますが、今回はフロントエンドで JS の記述量が多くなることが明らかだったので、 React + TypeScript の採用を検討し始めました。

また、ユーザーの権限に応じて機能が変わったり、UI も変わるので各状態ごとの UI を把握しにくくなることが課題として上がっていました。こちらの課題には Storybook が解決してくれそうで、かつ React とも相性が良かったので、こちらも React 採用のモチベーションに繋がりました。

### Rails の開発に React を取り入れる方法

これに関しては [mastodon](https://github.com/mastodon/mastodon) を参考にしました。mastodon では React と haml が混在していて、 React で UI を構築する場合は、 `mastodon` という `id` 属性を持つ DOM を記述した HTML をレンダリングし、その DOM 要素にコンポーネントを追加する、という方針をとっていました。

このやり方は React の採用以外にあまり新しいことを取り入れたくなかった私にとってはベストな方針でした。どうしたかというと、 `javascript` フォルダに `src` というフォルダを作成し、

そこに下記の `index.tsx` を作成し、 `application.ts` で `src` を import するだけで画面に `Hello, World!` が表示されます！

```tsx
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

```tsx
const App = () => {
	return <div>Hello, World!</div>;
}
```

とりあえず React で UI を構築出来る準備はこれで完了なのですが、フォルダの設計や状態管理などまだまだ考えないといけないことがたくさんあります。これらの課題に対してどう対応したのか、これから書いていこうと思います。

### フォルダ設計

一番悩んだ所です！というか今も悩んでます(笑)

私は `rails new` した時に生成される `javascript` フォルダを `frontend` に変更して、

さらに `src` を作成し、そこに React のコードをまとめることにしました。

つまり、最初は空っぽの状態から開発が始まるので、どうやってフォルダを分けて行けば良いかわかりませんでした。

フロントエンド の設計について語られている記事はたくさんありますが、多くの記事から参照されているリポジトリがありました。フロントエンド界隈では有名なようですが、 [bulletproof-react](https://github.com/alan2207/bulletproof-react) です。こちらは「package by feature」と呼ばれる設計手法についてまとめられています。

package by feature を採用して得られるメリットとしては、機能ごとにフォルダを分けるので、どこを読めば良いのか把握しやすく、またフォルダを開いたときに大量のファイルが出てくることを避けられると思いました。これは自分が求めていたものなので、まず上記のリポジトリの構成をベースにフォルダ設計を考えていくことにしました。

詳しくはこちらを参考にしてください。 https://github.com/alan2207/bulletproof-react/blob/master/docs/project-structure.md

そして、最終的にフォルダ構成はこんな感じになりました。全てを同じ構造にした訳ではなくプロジェクトの規模感などを考慮して一部アレンジしています。

```
.
└── src
    ├── App.tsx
    ├── __mocks__
    ├── components
    ├── **features**
    ├── hooks
    ├── index.tsx
    ├── lib
    ├── **pages**
    ├── providers
    ├── routes
    ├── test
    ├── types
    └── utils
```

ほとんどが**「**Project Structure」の章で解説されている通りなのですが、一部アレンジを加えた部分があるので、そこだけ解説していこうと思います。

| フォルダ | 役割 |
| --- | --- |
| pages | `#root` DOM 配下に追加されるコンポーネント |
| features | 機能単位で分割されたコード。  例）Calendar, Reservation |

`pages` 配下のコンポーネントは `features` と `components` のファイルで構成され、 `#root` 下に展開されるので、これが最終的なアウトプットになります。 また `pages` に関しては必ず Story を描くようにしていました。 Storybook に関しては後述します。

```
.
├── todos
│   ├── index.stories.tsx
│   └── index.tsx
└── users
    ├── index.stories.tsx
    └── index.tsx
```

次に`features` フォルダの中身の解説です。

| components | 特定の機能のためのコンポーネント 例) CreateTodo.tsx, DeleteTodo.tsx… |
| --- | --- |
| api | API リクエストを行う hooks。 TanStack Query を使用しています。 |
| schemas | react-hook-form で使用する zod で定義したスキーマ |
| types | OpenAPI で自動生成された型 |

`features` の中身は `components` と `api` は bulletprofe-react を完全にパクってて、 `components` には フォームやAPI から取得したデータの一覧を表示するコンポーネントが、

`api` には API リクエストのための関数が置かれていて、 TanStack Query を採用しています。

`schemas` はフォームの実装に使用する [zod](https://github.com/colinhacks/zod) で書かれたスキーマを置きました。

```tsx
const todoSchema = z.object({
	title: z.string(),
	content: z.string(),
});

type TodoSchema = z.infer<typeof todoSchema>;
```

「bulletproof-react」では `api` に書かれているのですが、フォームが巨大だったので別のフォルダに分けることにしました。

`types` には [openapi-typescript](https://github.com/openapi-ts/openapi-typescript/tree/main/packages/openapi-typescript) で生成された型をインポートしています。

```tsx
import type { components } from '@/swagger';

export type Todo = components['schemas']['todo'];
```

API のレスポンスの型は OpenAPI を使用して自動生成しています。