# CPT Sticky Posts

カスタム投稿タイプに「先頭に固定」機能を追加し、REST APIでの取得に対応するWordPressプラグイン。

## 機能

- カスタム投稿タイプの編集画面に「先頭に固定」チェックボックスを追加
- REST APIレスポンスに `sticky` フィールドを追加
- REST APIクエリパラメータ `sticky` と `sticky_first` を追加
- 投稿一覧画面に固定状態を表示

## インストール

1. `cpt-sticky-posts` フォルダを `/wp-content/plugins/` にアップロード
2. WordPress管理画面の「プラグイン」から有効化

## 使い方

### 管理画面

カスタム投稿タイプの編集画面のサイドバーに「先頭に固定」メタボックスが表示されます。チェックを入れて保存すると、その投稿が固定投稿として設定されます。

### REST API

#### レスポンスに含まれるフィールド

各投稿のレスポンスに `sticky` フィールド（boolean）が追加されます。

```json
{
  "id": 123,
  "title": { "rendered": "サンプル投稿" },
  "sticky": true,
  ...
}
```

#### クエリパラメータ

| パラメータ | 型 | 説明 |
|-----------|------|------|
| `sticky` | boolean | `true`: 固定投稿のみ取得、`false`: 非固定投稿のみ取得 |
| `sticky_first` | boolean | `true`: 固定投稿を先頭に表示（その後は通常の日付順） |

#### 使用例

```bash
# 固定投稿のみ取得
GET /wp-json/wp/v2/your_post_type?sticky=true

# 非固定投稿のみ取得
GET /wp-json/wp/v2/your_post_type?sticky=false

# 固定投稿を先頭に、残りは日付順で取得
GET /wp-json/wp/v2/your_post_type?sticky_first=true

# 組み合わせ例：固定投稿を先頭に、カテゴリーでフィルタリング
GET /wp-json/wp/v2/your_post_type?sticky_first=true&categories=5
```

### 対象の投稿タイプを制御

デフォルトでは `show_in_rest = true` かつ `public = true` のカスタム投稿タイプすべてが対象になります。

特定の投稿タイプのみを対象にしたい場合は、以下のフィルターを使用します：

```php
// functions.php または独自プラグインに追加
add_filter( 'cpt_sticky_posts_target_types', function( $post_types ) {
    // 'news' と 'event' のみを対象にする
    return [ 'news', 'event' ];
} );
```

## JavaScript/フロントエンドでの使用例

```javascript
// 固定投稿を先頭に取得
const response = await fetch('/wp-json/wp/v2/news?sticky_first=true&per_page=10');
const posts = await response.json();

// 固定投稿のみ取得
const stickyPosts = await fetch('/wp-json/wp/v2/news?sticky=true').then(r => r.json());

// 投稿のstickyフラグを更新（認証が必要）
await fetch('/wp-json/wp/v2/news/123', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-WP-Nonce': wpApiSettings.nonce
  },
  body: JSON.stringify({ sticky: true })
});
```

## 技術詳細

### 保存されるメタキー

- キー名: `_cpt_is_sticky`
- 値: `true` または `false` (boolean)

### フック

| フック | 種類 | 説明 |
|-------|------|------|
| `cpt_sticky_posts_target_types` | filter | 対象のカスタム投稿タイプを変更 |

## 注意事項

- CPT UIで作成したカスタム投稿タイプは、「REST APIで表示」が有効になっている必要があります
- `sticky_first` は `meta_query` を使用するため、大量の投稿がある場合はパフォーマンスに影響する可能性があります

## 要件

- WordPress 5.0以上
- PHP 7.4以上

## ライセンス

GPL v2 or later
