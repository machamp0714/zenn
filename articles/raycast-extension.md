---
title: "pending"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Raycast", "React"]
published: false
---

### 実装編

基本的に Raycast 公式が用意した Component を活用して実装していくことになります。

[User Interface | Raycast API](https://developers.raycast.com/api-reference/user-interface)

外部サービスと連携する必要がある拡張機能を実装する場合、API キーを入力してもらう必要がありますが、 [p**references](https://developers.raycast.com/information/manifest#preference-properties)** を `package.json` に記述するとコマンドを実行する際に↓の画面が出てきます。

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

preference は `getPreferenceValues` を使用して取得できます。

```tsx
interface Preferences {
  accessToken: string;
}

const preferences = getPreferenceValues<Preferences>();

console.log(preferences.accessToken);
```

API からデータを取得する場合も Raycast が用意してくれた `useCachedPromise` という hooks を使用して実装できます。 `stale-while-revalidate` 戦略に従うので Tanstack Query を使ったことがある方には馴染みあるインターフェースかと思います。

[useCachedPromise | Raycast API](https://developers.raycast.com/utilities/react-hooks/usecachedpromise)

関数を再実行したい場合、例えばデータを新規で作成して一覧データのキャッシュを無効化したいときは、 `revalidate` が使用できます。

データの作成に失敗したときや、フェッチに失敗したときのユーザーへの通知ですが、こちらは

[Toast](https://developers.raycast.com/api-reference/feedback/toast) を利用します。

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

API と連携する必要がある拡張機能はこの辺の Raycast API を組み合わせて実装を進めていくことになると思います。

他にも [Apple Script](https://developers.raycast.com/utilities/functions/runapplescript) や macOS の [MenuBar](https://developers.raycast.com/api-reference/menu-bar-commands) を操作できたり、Raycast が機能を提供してくれているので、目的に合ったものを探してみてください。

---

### 公開準備編

コマンドの実装が完了したら次は公開するための準備を進めていきます。

基本的には [こちら](https://developers.raycast.com/basics/prepare-an-extension-for-store) のドキュメントに書かれていることを実施すれば問題なく公開できます。

私の場合は特に何か指摘されることもなく approve されました。

https://github.com/raycast/extensions/pull/16178

こちらが実際の PR です。

ここからは実際に私が公開するためにやったことをまとめていきます。

#### 拡張機能のタイトルとコマンドの命名

まずタイトルとコマンドは Apple Style Guide に従う必要があるのですが、ちょっと長いです。

[C](https://support.apple.com/ja-jp/guide/applestyleguide/apsgb744e4a3/web)

全部把握しなくても公式のドキュメントに記載されてあるような、拡張機能のタイトルには「動詞よりも名刺を使う」「コマンドが少ない場合は一般的な命名にしない」やコマンドは「動詞+名詞」で構成するなど、この辺りのルールに気を付けていれば問題ないかと思います。

私は拡張機能のタイトルは

- コマンドの種類が少ない（打刻の開始/停止しかコマンドがない）
- 今後もコマンドを追加する予定がない

ことを考慮して「TimeCrowd」ではなく「TimeCrowd Tracker」にしました。

また、今回実装したコマンドは１つで「Start/Stop Time Entry」としています。

#### アイコン

アイコンは自作せず、Raycast が用意してくれたアイコンメーカーを利用しました。

特にこだわりがない場合はこちらで作成すれば良いと思います。

[Icon Maker by Raycast](https://ray.so/icon)

#### スクリーンショット

公式は最低でも3種用意することを推奨しているので、今回3枚撮りました。

スクリーンショットは Raycast の Window Capture を使用して撮れます。

まず、Raycast の Setting を開き、「Advanced」タブを開き、Window Capture でショートカットキーを指定します。

![スクリーンショット 2025-02-01 12.25.17.png](attachment:fa37e28f-dd56-4bb3-be47-7083ff213186:スクリーンショット_2025-02-01_12.25.17.png)

あとは、コマンドを開いてショートカットキーを呼び出せば Raycast 用のスクリーンショットを撮ることができます。[壁紙](https://www.raycast.com/wallpapers) もRaycast が用意してくれてます。

#### README

README にはAPI キーが必要だったので、それを取得する方法と、コマンドの簡単な説明を記載しました。特にスクショなどは用意する必要はなくテキストだけで公開できました。英語で記述する必要がありますが、これも Claude など AI に翻訳して貰えば良いと思います。

```markdown
## Getting Started

To use this extension, you need your TimeCrowd API Token.

1. Go to https://timecrowd.net/oauth/applications
2. Click "New Application"
3. In the **Name** field, enter a name of your choice for the application.
4. In the **Redirect URI** field, enter `urn:ietf:wg:oauth:2.0:oob`.
5. Click "Submit" then click "New token" then click "Issue"
6. Copy the token
```

公開するためにやったことの紹介は以上となります。アイコン、スクショも自前で用意する必要がなく Raycast が提供しているものを利用するだけなのでかなり楽でした。基本的に命名だけ注意してあとは README だけ丁寧に書いておけばレビューは通るのかなーと思ってます。

### さいごに

Raycast の拡張機能は Raycast API を組み合わせるだけで簡単に実装できるので、今後も何かやりたいことが出てきたら作ってみようかな、と思いました。

最後まで読んで頂きありがとうございます