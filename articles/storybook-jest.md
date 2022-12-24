---
title: "Storybook"
emoji: "🌊"
type: "tech"
topics: ["storybook", "jest", "typescript"]
published: false
---

この記事は何を解決するのか？
自分と似た様な構成で開発している人のためになれば良い

対象
- 最近 storybook をプロジェクトに導入した人
- これから jest を書こうとしている人

1. はじめに
2. 読者対象
3. jest のテストを書き直す => エラーが発生する
4. 解決方法を試す
5. エラーが発生するの解消方法を解説する(主題)
   1. jest.config.js の修正
   2. decorators.js
   3. tailwind.cssをモックする
   4. tsconfig.json を準備する => 書かなくても良いかも
6. 再度、テストがパスすることを確認

---

### Story を再利用して jest を記述する

まず [@storybook/testing-react](https://github.com/storybookjs/testing-react) をインストールします。
```shell
# npm の場合
$ npm install --save-dev @storybook/testing-react

# yarn の場合
$ yarn add -D @storybook/testing-react
```

早速 jest で Story を利用する様に修正してみます。

```typescript
import React from 'react';
import { render, within } from '@testing-library/react';
import { composeStories } from '@storybook/testing-react';
import '@testing-library/jest-dom';

import { TestStory } from './test.stories.tsx';

describe('test', () => {
  const { FilledSuccess } = composeStories(stories);

  describe('リクエストが成功した時', () => {
    it('アラートが表示されること', () => {
      const { container } = render(<FilledSuccess />);
      const canvas = within(container);

      await FilledSuccess.play();
      expect(screen.findByText('タスクが作成されました')).toBeInTheDocument();
    });
  });
});
```

Storybook をレンダリングする際に
```javascript
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const client = new QueryClient();

export const Decorator = (Story) => (
  <QueryClientProvider client={client}>
    <Story />
  </QueryClientProvider>
);
```
この様なデコレータを設定していたので、テストを実行すると次のエラーが発生しました。
`@storybook/testing-react` のREADMEを読むと
> If you have global decorators/parameters/etc and want them applied to your stories when testing them, you first need to set this up. You can do this by adding to or creating a jest setup file:
という記述が見つかるので、こちらを試してみます。

```javascript
import { setGlobalConfig } from '@storybook/testing-react';
import * as globalStorybookConfig from './.storybook/preview'; // path of your preview.js file

setGlobalConfig(globalStorybookConfig);
```

ここで `preview.js` の内訳を提示するのが良さそう。

を新たに作成し、
```javascript
/** @type {import('ts-jest').JestConfigWithTsJest} */

module.exports = {
  roots: ["<rootDir>/app/javascript/src"],
  preset: 'ts-jest',
  testMatch: [
    "**/__tests__/**/*.+(ts|tsx|js)",
    "**/?(*.)+(spec|test).+(ts|tsx|js)"
  ],
  testEnvironment: 'jsdom',
  setUpFiles: ['./setup.jest.js']
};
```
`jest.config.js` で読み込みます。

再度テストを実行すると、こちらのエラーが発生しました。
```
Jest encountered an unexpected token

Jest failed to parse a file. This happens e.g. when your code or its dependencies use non-standard JavaScript syntax, or when Jest is not configured to support such syntax.

Out of the box Jest supports Babel, which will be used to transform your files into valid JS based on your Babel configuration.

Details:

/Users/ooidetatsuya/repo/rails7-template/app/javascript/src/test/setup.jest.js:1
({"Object.<anonymous>":function(module,exports,require,__dirname,__filename,jest){import { setGlobalConfig } from '@storybook/testing-react';
                                                                                      ^^^^^^

SyntaxError: Cannot use import statement outside a module
```

`presets` を変更し `allowJs` を `true` に変更すると行ける。

`Node.js` では `import/export` 構文を使用することが出来ないので、
Jest は `setup.jest.js` の解析に失敗している様です。

https://kulshekhar.github.io/ts-jest/docs/guides/esm-support
こちらの記事によると `ts-jest` は ESM をサポートしているので、 `ts-jest` が
提供している `presets` を利用するように `jest.config.js` を修正します。

```diff js:jest.config.js
/** @type {import('ts-jest').JestConfigWithTsJest} */

module.exports = {
  roots: ["<rootDir>/app/javascript/src"],
- preset: 'ts-jest',
+ preset: 'ts-jest/presets/js-with-ts-esm',
  testMatch: [
    "**/__tests__/**/*.+(ts|tsx|js)",
    "**/?(*.)+(spec|test).+(ts|tsx|js)"
  ],
  testEnvironment: 'jsdom',
  setUpFiles: ['./setup.jest.js']
};
```
この `preset` を使うには `tsconfig.json` で `allowJs` を `true` にする必要があります。
``` diff json:tsconfig.json
{
  "compilerOptions": {
    "target": "es2016",
    "lib": [],
    "jsx": "react",
-   "allowJs": false,
+   "allowJs": true,
  },
  "exclude": ["node_modules"]
}
```
再度テストを実行します。

```
Details:

/Users/ooidetatsuya/repo/rails7-template/.storybook/preview.js:1
({"Object.<anonymous>":function(module,exports,require,__dirname,__filename,jest){import '../app/assets/stylesheets/application.tailwind.css';
                                                                                      ^^^^^^

SyntaxError: Cannot use import statement outside a module
```
次は `tailwind.css` を import している所でエラーが発生してしまいました。
どうやら jest は CSS や画像も js としてインポートしてしまうのでエラーが発生する様です。

[jest-transform-stub](https://github.com/eddyerburgh/jest-transform-stub) を使って CSSや画像ファイルをモック化
することでこちらのエラーには対応します。

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
これで再度テストを実行した所問題なくパスしました。