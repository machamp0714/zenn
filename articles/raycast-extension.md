---
title: "【Raycast Extension】TimeCrowd 打刻ツールを自作した"
emoji: "🐬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Raycast", "React"]
published: false
---

これまでランチャーアプリを使ったことがなかったのですが、Raycast を試したところ、その使い勝手の良さにすっかりハマってしまいました。今では、Magnet や Clipy で行なっていた作業を Raycast に移行したり、Obsidian へのメモを Raycast から直接行なえるようにするなど、さまざまなアクションを Raycast を起点として実行しています。

そこで、普段業務で使用している TimeCrowd という時間管理ツールでの打刻も Raycast から実行できるようにしたいと考え、今回拡張機能を作ることにしました。

## 作ったもの

![](/images/timecrowd-tracker.png)

「Start/Stop Time Entry」というコマンドだけ実装しました。現状、

1. 1週間分のタスクの一覧
2. 打刻の開始・停止
3. 新しいタスクの追加

のみ Raycast から実行できます。

https://www.raycast.com/machamp0714/timecrowd-tracker

## 実装

Raycast 拡張機能の開発は、Raycast アプリケーションに内蔵された「Create Extension」コマンドを実行するだけで開始できます。開発環境には `eslint` と `prettier` があらかじめ組み込まれているため、すぐにコーディングに取り掛かることができます。

拡張機能の UI は、Raycast が提供する Component を活用して構築します。外部 API との連携が不要な拡張機能であれば、下記のドキュメントを参照するだけでほぼ実装が完了すると思います。

https://developers.raycast.com/api-reference/user-interface

今回は TimeCrowd の API を使用するので、どう連携するかをまとめいきます。

### API キーの登録

外部サービスと連携する必要がある拡張機能を実装する場合、API キーを入力してもらう必要があります。Raycast では、package.json に **[preferences](https://developers.raycast.com/information/manifest#preference-properties)** を記述することで、コマンド実行時に API キーの入力画面を表示させることができます。

```json
{
  "preferences": [
    {
      "description": "The access token for your TimeCrowd account.",
      "name": "accessToken",
      "placeholder": "Can be found at https://timecrowd.net/oauth/applications",
      "required": true,
      "title": "TimeCrowd Access Token",
      "type": "password"
    }
  ]
}
```

上記の例では、accessToken という名前の API キーを必須項目として定義しています。type を password に設定することで、入力された API キーがマスク表示されます。

![](/images/tc-api-key.png)

入力した API キーは `getPreferenceValues` を使用して取得できます。

```tsx
interface Preferences {
  accessToken: string;
}

const preferences = getPreferenceValues<Preferences>();

console.log(preferences.accessToken);
```

### API 連携

TimeCrowd API からデータを取得する場合、Raycast が提供する `useCachedPromise` という React Hook を活用することで、`stale-while-revalidate` 戦略に基づいたデータ取得とキャッシュの仕組みを実現できます。

https://developers.raycast.com/utilities/react-hooks/usecachedpromise

`useCachedPromise` で取得したデータを更新したい場合、`revalidate` 関数を使用します。例えば、TimeCrowd で新しいタスクを作成した場合、一覧データを更新する必要がありますが、このとき、revalidate を実行することで、キャッシュを無効化し、最新のデータを取得できます。

### Toast

データの作成に失敗した場合や、API との連携に失敗した場合など、ユーザーへの通知には Raycast の Toast API を活用します。

https://developers.raycast.com/api-reference/feedback/toast

```tsx
const { handleSubmit, itemProps, setValue, setValidationError } = useForm<TimeEntryForm>({
  async onSubmit(values) {
    // 省略
    try {
      await createTimeEntry(values);
      showToast(Toast.Style.Success, `Started Time Entry`);
    } catch (error) {
      showToast(Toast.Style.Failure, "Failed to create Time Entry");
    }
  },
});
```

## 公開準備

コマンドの実装が完了したら、拡張機能を公開するための準備に取り掛かります。
Raycast 拡張機能の公開は、基本的に公式ドキュメントに記載されている手順に従って行ないます。

https://developers.raycast.com/basics/prepare-an-extension-for-store

私の場合は、特に指摘を受けることもなくスムーズに承認されました。
実際の PR です↓

https://github.com/raycast/extensions/pull/16178

ここでは、私が実際に行なった公開準備の手順についてまとめます。

### アイコン

拡張機能のアイコンは、デフォルトのものではなくちゃんと用意する必要があります。
とはいっても自前で作成する必要はなく、Raycast が提供するアイコンメーカーを利用すると、簡単にアイコンを作成できます。特にこだわりがない場合は、こちらを利用することをおすすめします。

https://ray.so/icon

### スクリーンショット

Raycast 拡張機能の公開にあたっては、最低でも 3 つのスクリーンショットを用意することが推奨されており、スクリーンショットは、Raycast に内蔵された Window Capture 機能を利用して撮影できます。

1. Raycast の設定画面を開き、「Advanced」タブを選択します。
2. Window Capture のショートカットキーを設定します。
3. 拡張機能を実行し、ショートカットキーを押すことで、Raycast のスクリーンショットを撮影できます。

![](/images/window-capture.png)

### README

README には API キーの取得方法とコマンドの簡単な説明だけ記載しました。文章は Claude に翻訳してもらったものをそのまま記載しています。

https://github.com/machamp0714/timecrowd-tracker/blob/main/README.md

公開するためにやったことの紹介は以上となります。アイコンやスクショも Raycast が提供しているものを利用するだけなのでかなり楽でした。拡張機能のタイトルとコマンドの命名だけ注意していれば、割と簡単に公開まで行けると思います。

公開準備が整ったら `npm run publish` を実行してレビューを待ちます。私は2週間程度で公開されました。

## さいごに

拡張機能ですが、Raycast API を組み合わせるだけで簡単に実装できたので、今後も何かやりたいことが出てきたら作ってみようかな、と思いました。最後まで読んでいただきありがとうございます🙇‍♂️