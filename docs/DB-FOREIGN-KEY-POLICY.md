# 外部キー削除時挙動(ON DELETE)ポリシー

## 目的

DBML(ER図)では表現しきれない「親レコードが削除された際、子レコードをどう扱うか」の方針を明文化する。実装時(DDL・Spring Data JPA)は本ドキュメントに従う。

## 基本方針

- **履歴として保持すべきデータ(ユーザー・写真)は物理削除せず、論理削除(フラグ)で対応する。これに伴い、関連する外部キーは`RESTRICT`とし、そもそも物理削除自体を起こさせない**
- **紐づく親が消えれば意味を失う、使い捨て・従属的なデータ(カート明細・注文明細・DLトークン)は`CASCADE`で自動的に削除する**

## 一覧

| 外部キー | 親テーブル | ON DELETE | 理由 |
|---|---|---|---|
| `photos.seller_id` | `users` | RESTRICT | 出品者を物理削除すると、購入済みユーザーの参照先が壊れる。退会は`users.is_active = false`で対応 |
| `orders.user_id` | `users` | RESTRICT | 注文履歴の保護。退会は`users.is_active = false`で対応、ユーザー自体は削除しない |
| `cart_items.cart_id` | `carts` | CASCADE | カートが削除されれば、その中身も一緒に消えて自然 |
| `cart_items.photo_id` | `photos` | RESTRICT | 写真は物理削除せず`photos.is_deleted = true`で対応するため、実質発動しない想定 |
| `order_items.order_id` | `orders` | CASCADE | 注文と明細は一心同体。ただし注文自体は基本的に削除せず、`status`を`CANCELLED`等に変更するのが実運用 |
| `order_items.photo_id` | `photos` | RESTRICT | 購入済み写真の情報を消すと注文履歴が壊れる。`photos.is_deleted`で対応 |
| `download_tokens.order_item_id` | `order_items` | CASCADE | 明細が消えれば、対応するDLトークンも不要になるため |

## 削除の代替手段(論理削除)

物理削除を避けるため、以下のフラグで「削除相当」の状態を表現する。

| テーブル | カラム | 意味 |
|---|---|---|
| `users` | `is_active` | `false`で退会・無効化済み扱い。ログイン不可にする |
| `photos` | `is_deleted` | `true`で出品終了・非表示扱い。一覧・検索からは除外するが、購入済みユーザーは引き続き参照可能 |

## Spring Data JPAでの指定例

```java
@ManyToOne
@JoinColumn(
    name = "seller_id",
    nullable = false,
    foreignKey = @ForeignKey(
        name = "fk_photos_seller",
        foreignKeyDefinition = "FOREIGN KEY (seller_id) REFERENCES users(id) ON DELETE RESTRICT"
    )
)
private User seller;
```

`CASCADE`にする場合は、`ON DELETE RESTRICT`の部分を`ON DELETE CASCADE`に置き換える。

## 改訂

方針を変更する場合は、変更内容と理由をこのファイルに追記し、Gitリポジトリにコミットして履歴を残す。

| 日付 | 内容 |
|---|---|
| 初版 | DBMLレビューを踏まえ、CASCADE/RESTRICTの使い分け方針を整理 |
