---
title: "StorybookをJestで再利用する"
emoji: "🌊"
type: "tech"
topics: ["storybook", "jest", "typescript"]
published: false
---

Jestでテストを書くとき、Storybookで設定したデコレーターと同じ設定を記述しなければ
いけず、なんとかならないだろうかと思っていたところ、[@storybook/testing-react](https://github.com/storybookjs/testing-react) を利用すると
StoryをJestで再利用することを知り早速試してみました。
ただ所々エラーが発生したので解消方法を残しておこうと思います。

**対象**
1. 最近Storybookをプロジェクトに導入した方
2. Jestでフロントエンドのテストを頑張ろうと決意した方

## Storyを再利用してjestを記述する

まず`@storybook/testing-react`をインストールします。
```shell
# npm の場合
$ npm install --save-dev @storybook/testing-react

# yarn の場合
$ yarn add -D @storybook/testing-react
```

そして`@storybook/testing-react`のREADMEを参考にJestでストーリーを再利用する様に
テストを修正します。
```ts:index.spec.tsx
import React from 'react';
import { render, within } from '@testing-library/react';
import { composeStories } from '@storybook/testing-react';
import '@testing-library/jest-dom';

import * as stories from './index.stories.tsx';

describe('TaskForm', () => {
  it('メッセージが表示されること', async () => {
    const { Success } = composeStories(stories);
    const { container } = render(<Success />);
    const canvas = within(container);
    Success.play({ canvasElement: container });

    expect(await canvas.findByText('タスクが作成されました')).toBeInTheDocument();
  });
});
```
:::details TaskFormコンポーネント
```tsx: index.tsx
import React from 'react';

export const TaskForm = () => {
  // 省略

  return (
    <div>
      <h1>タスク作成</h1>
      <form onSubmit={handleSubmit} />
        <div>
          <label htmlFor="title">タスク名</label>
          <input id="title" type="text" placeholder="タスク名を入力してください" />
        </div>
        <button type="submit">作成</button>
      </form>
    </div>
  );
};
```
:::
:::details TaskFormストーリー
```tsx: index.stories.tsx
import { ComponentMeta, ComponentStoryObj } from '@storybook/react';
import { userEvent, within } from '@storybook/testing-library';

import { TaskForm } from './TaskForm';

type Story = ComponentStoryObj<typeof TaskForm>;

export default {
  component: TaskForm,
} as ComponentMeta<typeof TaskForm>;

export const Default: Story = {}

export const Success: Story = {
  ...Default,
  play: ({ canvasElement }) => {
    const canvas = within(canvasElement);

    userEvent.type(canvas.getByLabelText('タスク名'), 'title');
    userEvent.click(canvas.getByText('作成'));
  },
};
```
:::
Storybookではデコレーターを使用しているのでセットアップファイルを`jest.config.js`で読み込むように構成を変更します。
```js:setup.jest.js
import { setGlobalConfig } from '@storybook/testing-react';
import * as globalStorybookConfig from './.storybook/preview';

setGlobalConfig(globalStorybookConfig);
```
:::details preview.js
```js:.storybook/preview.js
import React from 'react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import '../app/assets/stylesheets/application.tailwind.css';

export const parameters = {
  actions: { argTypesRegex: '^on[A-Z].*' },
  controls: {
    matchers: {
      color: /(background|color)$/i,
      date: /Date$/,
    },
  },
};

const client = new QueryClient();
const BaseDecorator = (Story) => (
  <QueryClientProvider client={client}>
    <Story />
  </QueryClientProvider>
);

export const decorators = [BaseDecorator];
```
:::
```js:jest.config.js
/** @type {import('ts-jest').JestConfigWithTsJest} */

module.exports = {
  roots: ["<rootDir>/app/javascript/src"],
  preset: 'ts-jest',
  testEnvironment: 'jsdom',
  setUpFiles: ['./setup.jest.js']
};
```
この状態でテストを実行すると..。
```
Jest encountered an unexpected token

Jest failed to parse a file. This happens e.g. when your code or its dependencies use non-standard JavaScript syntax, or when Jest is not configured to support such syntax.

Out of the box Jest supports Babel, which will be used to transform your files into valid JS based on your Babel configuration.

Details:

/Users/machmap/repo/test/app/frontend/src/test/setup.jest.js:1
({"Object.<anonymous>":function(module,exports,require,__dirname,__filename,jest){import { setGlobalConfig } from '@storybook/testing-react';
                                                                                      ^^^^^^

SyntaxError: Cannot use import statement outside a module
```
こちらのエラーが発生しました。`Node.js` では `import/export` 構文を使用することが
できないので、Jestは `setup.jest.js` の解析に失敗している様です。

## ts-jestのpresetを変更する

https://kulshekhar.github.io/ts-jest/docs/getting-started/presets
 `ts-jest` はESMに変換してくれる`presets`を提供してくれているので、そちらを使用する様に
 `jest.config.js`を修正します。
```diff js:jest.config.js
/** @type {import('ts-jest').JestConfigWithTsJest} */

module.exports = {
  roots: ["<rootDir>/app/javascript/src"],
- preset: 'ts-jest',
+ preset: 'ts-jest/presets/js-with-ts-esm',
  testEnvironment: 'jsdom',
  setUpFiles: ['./setup.jest.js']
};
```
:::message
この`preset`を使用するにはtsconfig.jsonで`allowJs`を`true`に変更する必要があります
:::
再度テストを実行します。

```
 Details:

/Users/machamp/repo/test/app/assets/stylesheets/application.tailwind.css:1
({"Object.<anonymous>":function(module,exports,require,__dirname,__filename,jest){@tailwind base;
```
今度は`tailwind`をimportしているところでエラーが発生しました。

## CSSではなくモックを読み込む様に修正する

[jest-transform-stub](https://github.com/eddyerburgh/jest-transform-stub) を使って`tailwind`をモック化することでこのエラーには対応できます。

```diff js:jest.config.js
/** @type {import('ts-jest').JestConfigWithTsJest} */

module.exports = {
  roots: ["<rootDir>/app/javascript/src"],
  preset: 'ts-jest/presets/js-with-ts-esm',
  testMatch: [
    "**/__tests__/**/*.+(ts|tsx|js)",
    "**/?(*.)+(spec|test).+(ts|tsx|js)"
  ],
  testEnvironment: 'jsdom',
+ transform: {
+   ".+\\.(css|styl|less|sass|scss|png|jpg|ttf|woff|woff2)$": "jest-transform-stub"
+ },
  setUpFiles: ['./setup.jest.js']
};
```
を追加します。
これで再度テストを実行したところ、テストがパスしました🎉
