---
title: GraphQL::Batchを使ってよくあるユースケースをTwitterを例に実装してみる
published: 2022-03-16
updated: 2022-03-16T10:00:00+09:00
url: https://product.plex.co.jp/entry/dataloader-graphql-batch
entry-id: tag:blog.hatena.ne.jp,2013:blog-plexjp-26006613789038539-13574176438068415851
author: ishitsukajun
edited: 2026-01-08T14:28:39+09:00
tags:
  - GraphQL
  - Rails
  - Ruby
---

[f:id:plexjp:20260108142827p:plain]

こんにちは、プレックスの [石塚](https://twitter.com/ij_spitz) です。

今回のブログでは、Ruby向けのデータローダーである [GraphQL::Batch](https://github.com/Shopify/graphql-batch) を使用して、よくアプリケーションで実装されるユースケースの実装方法をTwitterを例に紹介します。

[:contents]

## データローダーとは？

データローダーとはGraphQLに用意されているデータを一括取得するための仕組みです。

例えば以下のようにユーザーを5件取得して、そのユーザーのツイート一覧をデータローダーを使用しないで取得しようとすると下記のように6回（5 + 1回）のクエリが発生することになります。

```graphql
{
  users(first: 5) {
    id
    tweets {
      id
      content
    }
  }
}
```

```sql
SELECT id FROM users limit 5;
SELECT id, content FROM tweets WHERE id = 1;
SELECT id, content FROM tweets WHERE id = 2;
SELECT id, content FROM tweets WHERE id = 3;
SELECT id, content FROM tweets WHERE id = 4;
SELECT id, content FROM tweets WHERE id = 5;
```

これをデータローダーを使用することにより、以下のように2回のクエリでデータを取得できるようになります。

```sql
SELECT id FROM users limit 5;
SELECT id, content FROM tweets WHERE id IN (1, 2, 3, 4, 5);
```

## GraphQLにおけるデータローダーの必要性
GraphQLではオブジェクトがネストしており、再帰的に処理を読み出せる構造になっているため、N+1が起きやすいという特徴があります。

Railsに慣れている方ならば、それってpreloadやincludesを使用すればN+1を避けることができるのではないのか？と思うかもしれません。
しかしその方法で対処してしまうと関連するモデルの呼び出しが必要のないところでも、不必要にpreloadやincludesなどが走ってしまったり、再帰的に呼び出された場合にN+1を避けることが難しくなってしまいます。そのため、基本的にはデータローダーを使うのがベターです。

## 本エントリーで実装したいこと

以下の3つのユースケースを実装していきます。Twitterを例にしていますが、サブテーマとして書いているように、アプリケーションを作っていると似たような機能を作ることがよくあるのではないかと思います。

- 複数のユーザーのツイート一覧を取得する
  - ActiveRecordのリレーションをデータローダーに置き換える
- あるユーザーのツイート一覧をいいね数と一緒に取得する
  - GROUP BYでのカウントの実装
- あるユーザーのツイート一覧を自身がいいねしたかどうかと一緒に取得する
  - WHERE句などの条件指定をできるように拡張する

## 前準備

動作環境やライブラリのバージョンは下記のとおりです。

|  | バージョン |
| ---- | ---- |
| ruby | 3.0.2 |
| rails | 6.1.4 |
| graphql-ruby | 1.12.17 |
| graphql-batch | 0.4.3 |

Rubyにはgraphql-ruby標準のデータローダーもありますが、弊社では、スターの数が多く、使いやすいと感じた [GraphQL::Batch](https://github.com/Shopify/graphql-batch) を使っています。

以下のモデルファイルと対応するテーブルが存在すると仮定します。

```ruby
class User < ApplicationRecord
  has_many :tweets
  has_many :favorites
end

class Tweet < ApplicationRecord
  has_many :favorites
  belongs_to :user
end

class Favorite < ApplicationRecord
  belongs_to :user
  belongs_to :tweet
end
```

## 実装

### 複数のユーザーのツイート一覧を取得する（ActiveRecordのリレーションをデータローダーに置き換える）

まずは一番基本的なユースケースとして起こり得そうな、複数のユーザーのツイート一覧を取得するQueryから実装していきます。

N+1を引き起こすコードを書いてから、それをデータローダーを使った実装にリファクタリングしていくという流れで書いていきます。

**N+1を引き起こすコード**

```ruby
# app/graphql/types/query_type.rb

module Types
  class QueryType < Types::BaseObject
    include GraphQL::Types::Relay::HasNodeField
    include GraphQL::Types::Relay::HasNodesField

    field :users, [UserType], null: false do
      description 'ユーザー一覧をツイート一覧と共に取得する'
    end

    def users
      User.limit(5)
    end
  end
end
```

```ruby
# app/graphql/types/user_type.rb

module Types
  class UserType < Types::BaseObject
    field :id, ID, null: false
    field :tweets, [TweetType], null: false
  end
end
```

```ruby
# app/graphql/types/tweet_type.rb

module Types
  class TweetType < Types::BaseObject
    field :id, ID, null: false
    field :content, String, null: false
  end
end
```

これでN+1を引き起こすQueryの実装は完了です。幸か不幸かRailsにはアソシエーションという機能があるため、たったこれだけのコードでツイートとともにユーザーを取得するAPIを書けるのですが、N+1を引き起こしてしまいます。

**データローダーを使ったコード**

これをデータローダーを使ったコードに書き換えていきます。まずは `GraphQL::Batch` のインストール。

```zsh
$ gem install graphql-batch
```

次にスキーマで `GraphQL::Batch` を使うように宣言します。

```ruby
# app/graphql/my_schema.rb

class MySchema < GraphQL::Schema
  query MyQueryType
  mutation MyMutationType
  use GraphQL::Batch
end
```

`GraphQL::Batch` にはRailsのアソシエーションをよしなに読み込んでくれる [AssociationLoader](https://github.com/Shopify/graphql-batch/blob/master/examples/association_loader.rb) というクラスがサンプルで用意されています。
これと同じコードを `app/graphql/loaders/association_loader.rb` に記述します。

```ruby
# app/graphql/loaders/association_loader.rb

module Loaders
  class class AssociationLoader < GraphQL::Batch::Loader
    def self.validate(model, association_name)
      new(model, association_name)
      nil
    end

    def initialize(model, association_name)
      super()
      @model = model
      @association_name = association_name
      validate
    end

    def load(record)
      raise TypeError, "#{@model} loader can't load association for #{record.class}" unless record.is_a?(@model)
      return Promise.resolve(read_association(record)) if association_loaded?(record)
      super
    end

    # We want to load the associations on all records, even if they have the same id
    def cache_key(record)
      record.object_id
    end

    def perform(records)
      preload_association(records)
      records.each { |record| fulfill(record, read_association(record)) }
    end

    private

    def validate
      unless @model.reflect_on_association(@association_name)
        raise ArgumentError, "No association #{@association_name} on #{@model}"
      end
    end

    def preload_association(records)
      ::ActiveRecord::Associations::Preloader.new(records: records, associations: @association_name).call
    end

    def read_association(record)
      record.public_send(@association_name)
    end

    def association_loaded?(record)
      record.association(@association_name).loaded?
    end
  end
end
```

そして `UserType` から作成した `AssociationLoader` を使うように変更します。

```ruby
# app/graphql/types/user_type.rb

module Types
  class UserType < Types::BaseObject
    field :id, ID, null: false
    field :tweets, [TweetType], null: false

    def tweets
      Loaders::AssociationLoader.for(User, :tweets).load(object)
    end
  end
end
```

これでリレーションではなく、データローダーを使ってツイートを読み込んでくれるため、N+1を解消できます。

### あるユーザーのツイート一覧をいいね数と一緒に取得する（GROUP BYでのカウントの実装）

次にいいね数をカウントしてツイート一覧を返す実装です。

**N+1を引き起こすコード**

こちらもN+1を起こす実装から作っていきましょう。

```ruby
# app/graphql/types/query_type.rb

module Types
  class QueryType < Types::BaseObject
    field :tweets, [TweetType], null: false do
      description 'ツイート一覧をいいね数と共に取得する'
    end

    def tweets
      Tweet.limit(5)
    end
  end
end
```

```ruby
# app/graphql/types/tweet_type.rb

module Types
  class TweetType < Types::BaseObject
    field :id, ID, null: false
    field :content, String, null: false
    field :favorite_count, Int, null: false

    def favorite_count
      object.favorites.size
    end
  end
end
```

こちらで無事に？N+1クエリが発生します。

**データローダーを使ったコード**

次にデータローダーを作ってN+1を解消していきます。アソシエーション用のデータローダーは `GraphQL::Batch` で用意されていましたが、集計用のデータローダーは用意されていないため、アソシエーションを参考に自前で作っていきます。

```ruby
# app/graphql/loaders/record_count_loader.rb

module Loaders
  class RecordCountLoader < GraphQL::Batch::Loader
    def initialize(model, column: model.primary_key, distinct_column: nil)
      super()
      @model = model
      @column = column.to_s
      @column_type = model.type_for_attribute(@column)
      @distinct_column = distinct_column
    end

    def load(key)
      super(@column_type.cast(key))
    end

    def perform(keys)
      query(keys).each { |key, count| fulfill(key, count) }
      keys.each { |key| fulfill(key, 0) unless fulfilled?(key) }
    end

    private

      def query(keys)
        scope = @model
        scope.where(@column => keys)
             .group(@column)
             .distinct(@distinct_column)
             .count
      end
  end
end
```

そして `TweetType` で実装したデータローダーを使うようにします。


```ruby
# app/graphql/types/tweet_type.rb

module Types
  class TweetType < Types::BaseObject
    field :id, ID, null: false
    field :content, String, null: false
    field :favorite_count, Int, null: false

    def favorite_count
      Loaders::RecordCountLoader.for(Favorite,
                                     column: 'tweet_id',
                                     distinct_column: :tweet_id).load(object.id)
    end
  end
end
```

これでN+1を回避できます。データローダーを汎用的に使い回せる形で実装しておくと、タイプから呼び出すだけで実装が済むので便利ですね。

### あるユーザーのツイート一覧を自身がいいねしたかどうかと一緒に取得する（WHERE句などの条件指定をできるように拡張する）

最後に特定のユーザーのツイート一覧を自身がいいねしたかどうかと一緒に取得するクエリを考えます。

**N+1を引き起こすコード**

こちらもまずはN+1を起こすクエリから書いていきます。

```ruby
# app/graphql/types/query_type.rb

module Types
  class QueryType < Types::BaseObject
    field :user, UserType, null: false do
      description '特定のユーザーのツイート一覧を自身がいいねしたかどうかと一緒に取得する'
      argument :id, ID, required: true
    end

    def tweet(id:)
      Tweet.find(id)
    end
  end
end
```

`UserType` には先ほど作成した `AssociationLoader` でツイート一覧を読み込みます。

```ruby
# app/graphql/types/user_type.rb

module Types
  class UserType < Types::BaseObject
    field :id, ID, null: false
    field :tweets, [TweetType], null: false

    def tweets
      Loaders::AssociationLoader.for(User, :tweets).load(object)
    end
  end
end
```

`TweetType` に新しく `is_favorited` という自身がいいねしたかどうかを表すフィールドを追加します。

認証情報は `context` に入っていると仮定して、 自身のユーザーIDをそこから取得しています。

```ruby
# app/graphql/types/tweet_type.rb

module Types
  class TweetType < Types::BaseObject
    field :id, ID, null: false
    field :content, String, null: false
    field :favorite_count, Int, null: false
    field :is_favorited, Boolean, null: false

    def favorite_count
      Loaders::RecordCountLoader.for(Favorite,
                                     column: 'tweet_id',
                                     distinct_column: :tweet_id).load(object.id)
    end

    def is_favorited
      object.favorites.exists?(user_id: context[:user].id)
    end
  end
end
```

これでいいねしたかどうかの情報が取れるようになり、N+1が発生します。

**データローダーを使ったコード**

`RecordPresenceLoader` というデータローダーを新しく作成します。

`RecordCountLoader` との違いとしては引数として `where` を受け取れるようにしている点です。 `GraphQL::Batch` では `ActiveRecord` をベースにデータローダーを拡張していけるので、このような形でWHERE句を追加したり、JOIN句を追加していくことができます。

```ruby
# app/graphql/loaders/record_presence_loader.rb

module Loaders
  class RecordPresenceLoader < GraphQL::Batch::Loader
    def initialize(model, column: model.primary_key, where: nil)
      super()
      @model = model
      @column = column.to_s
      @column_type = model.type_for_attribute(@column)
      @where = where
    end

    def load(key)
      super(@column_type.cast(key))
    end

    def perform(keys)
      query(keys).each { |record| fulfill(record.public_send(@column), record.present?) }
      keys.each { |key| fulfill(key, false) unless fulfilled?(key) }
    end

    private

      def query(keys)
        scope = @model
        scope = scope.where(@where) if @where
        scope.where(@column => keys)
      end
  end
end
```

そして `RecordPresenceLoader` をタイプから使用します。

```ruby
# app/graphql/types/tweet_type.rb

module Types
  class TweetType < Types::BaseObject
    field :id, ID, null: false
    field :content, String, null: false
    field :favorite_count, Int, null: false
    field :is_favorited, Boolean, null: false

    def favorite_count
      Loaders::RecordCountLoader.for(Favorite, column: 'tweet_id', distinct_column: :tweet_id).load(object.id)
    end

    def is_favorited
      Loaders::RecordPresenceLoader.for(Favorite,
                                        column: 'tweet_id',
                                        where: {
                                          user_id: context[:user].id
                                        }).load(object.id)
    end
  end
end
```

これで自身がいいねしたかどうかの情報もN+1を起こさずに取得できるようになりました。

## 終わりに

今回の記事はデータローダーのライブラリの使い方という小ネタでしたが、普段GraphQLの実装をしていると、日本語で書かれている情報がまだまだ少ないと感じることが多いため、少しでも参考になればと思って書いてみました。

最後になりますが、プレックスでは[ソフトウェアエンジニア](https://dev.plex.co.jp/0cb2e596183f4a2c9ebfa8f3cccab6c2)、[フロントエンドエンジニア](https://dev.plex.co.jp/47c0210f11884e2fbc8d12f3caf8fb23)を募集しています。

少しでも興味を持っていただけた方は業務委託や副業からでも、ぜひご応募いただけると幸いです。

[https://dev.plex.co.jp/:embed:cite]


