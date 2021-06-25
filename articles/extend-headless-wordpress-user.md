---
title: "Shifter Headlessでユーザーメタデータを拡張する"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Wordpress", "Shifter"]
published: true
---

[Beyond Magazine](https://beyondmag.jp?utm_source=zenn) では Shifter Headless をプロダクト環境で使用しています。  
[(Shifter Static と Headless の違いについてはこちら)](https://www.getshifter.io/ja/explaining-the-difference-between-shifter-static-and-shifter-headless-ja/)
今回は Shifter Headless を導入した際に、プラグインに頼らずにどうやってユーザーメタデータを追加するかを紹介します。

## なぜユーザーメタデータを拡張する必要があるか

Beyond Magazine の強みとして、
**各界一流の識者やインフルエンサーたちが執筆するメディアである**
という点があります。
そのためコンテンツだけではなく、執筆者の方たちのプロフィールを充実させて魅力的に見せることも重要です。
Wordpress はデフォルトであれば氏名や自己紹介などの項目しか利用できません。
しかし、各 SNS や個人のウェブサイトも執筆者にとって重要なプロフィールです。
なので、カスタマイズによりこれらの項目を追加する必要があります。
通常の Wordpress であればプラグインを入れることによって拡張が可能かもしれません。
しかし、[以前の記事](https://zenn.dev/takasing/articles/first-custom-block)で紹介したように、 Shifter Headless では plugin のインストールができません。
なので今回も[Code Snippets](https://ja.wordpress.org/plugins/code-snippets/)プラグインを使ってメタデータを拡張していきます。

## やること

Wordpress を Headless CMS として利用し、GraphQL 経由でフロントエンドとコミュニケーションするためには以下の 2 つの作業が必要です。

1. Wordpress のプロフィール入力ページに拡張したい項目の入力欄を設ける
   1. フォームを追加する
   1. 保存した際にデータを更新する
1. WPGraphQL のスキーマを拡張する

これらを Code Snippets を利用して行います。

なお、完成形は[このリポジトリ](https://github.com/takasing/wordpress-extend-graphql-schema)に上げているので、Import して使ってください。

## プロフィール入力ページに拡張したい項目の入力欄を設ける

ここからはシンプルに snippets を記載していきます。
まずはフォーム自体を表示します。

```php:expand_profile_v1
function append_custom_fields ($user) {
	?>
	<table id="custom_user_field_table" class="form-table">
        <tr id="custom_user_field_row">
            <th>
                <label for="twitter">Twitter</label>
            </th>
            <td>
                <input type="text" name="twitter" id="twitter" value="<?php echo esc_attr( get_the_author_meta( 'twitter', $user->ID ) ); ?>" class="regular-text" /><br />
                <span class="description">@以下を入力</span>
            </td>
        </tr>
    </table>
<?php
}
add_action( 'show_user_profile', 'append_custom_fields' );
add_action( 'edit_user_profile', 'append_custom_fields' );
```

これを保存し、プロフィールページに行くと、下の方に入力フォームが追加されていることを確認できます。

![プロフィール入力欄を追加](https://storage.googleapis.com/zenn-user-upload/a49bd714605ac42939b6d51f.png)

しかし、このままだと保存ボタンを押してもデータ自体は保存されないので、データを保存する記述を追加します。

```php:expand_profile_v2
function append_custom_fields ($user) {
	?>
	<table id="custom_user_field_table" class="form-table">
        <tr id="custom_user_field_row">
            <th>
                <label for="twitter">Twitter</label>
            </th>
            <td>
                <input type="text" name="twitter" id="twitter" value="<?php echo esc_attr( get_the_author_meta( 'twitter', $user->ID ) ); ?>" class="regular-text" /><br />
                <span class="description">@以下を入力</span>
            </td>
        </tr>
    </table>
<?php
}
add_action( 'show_user_profile', 'append_custom_fields' );
add_action( 'edit_user_profile', 'append_custom_fields' );
function edit_additional_profile ($user_id) {
	if ( isset( $_POST['twitter'] ) ) {
        update_user_meta($user_id, 'twitter', $_POST['twitter']);
    }
}
add_action('profile_update', 'edit_additional_profile');
```

これでプロフィールページに追加したメタデータが Wordpress に保存されるようになります。

## WPGraphQL のスキーマを拡張する

Wordpress にメタデータは保存されましたが、このままではまだ WPGraphQL のスキーマに Twitter アカウント用のフィールドが存在しませんので拡張します。

```php:expand_gq
function add_custom_fields() {
  register_graphql_field('user', 'twitter', [
    'type' => 'String',
    'args' => [],
    'resolve' => function ($source) {
		$fields = get_user_meta($source->userId, 'twitter', true);
		if ( isset( $fields ) ) {
      		return $fields;
   		}
    	return null;
	}
  ]);
}

add_action( 'graphql_register_types', 'add_custom_fields');
```

![GraphiQLの実行結果](https://storage.googleapis.com/zenn-user-upload/8ec574602db01000ecdaa624.png)

## まとめ

Wordpress を拡張する際は、「`add_action`などを用いて特定の処理に hook して処理を追加する」というのを覚えておくと理解しやすいです。
後は少しの php を(血の涙を流しながら)書くだけです。
Action に関しては [Wordpress デベロッパー向けドキュメントの hooks](https://developer.wordpress.org/reference/hooks/)を参照すると、どういうイベントに引っ掛けたいかを決めやすいです。
Plugin についても同様で、Plugin が追加した Hook があるので、そこに処理を hook させるように実装します。

[Beyond Magazine](https://beyondmag.jp?utm_source=zenn) では開発メンバーを募集しているのでご興味ある方は Twitter の DM 下さい！
![](https://storage.googleapis.com/zenn-user-upload/umo6g8r6dcoj4pz2poy8srgbq8xn)

## 参考

- [Extending WPGraphQL for Custom Meta Boxes with React and Apollo in a WordPress Theme Tutorial](https://javascriptforwp.com/extending-wpgraphql-for-custom-meta-boxes-with-react-and-apollo-in-a-wordpress-theme-tutorial/)
- [WPGraphQL - register_graphql_field](https://www.wpgraphql.com/functions/register_graphql_field/)
- [Registering GraphQL Fields with Arguments](https://www.wpgraphql.com/2020/03/11/registering-graphql-fields-with-arguments/)
- [How to add custom fields to the schema?](https://github.com/wp-graphql/wp-graphql/issues/1097)
- [User Profile / Add Custom Fields](https://wordpress.stackexchange.com/questions/39285/user-profile-add-custom-fields)
