---
title: "How to GraphQLを試す(connecting nodes)"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GraphQL", "Ruby"]
published: true
---

# 前記事
この記事は、以下の続きです。

https://zenn.dev/tossy21/articles/d0bb29cb0fcd2b

# ノードの接続

## リンクへの投票

ユーザがリンクに投票できるようにしていきます。

```bash
% bundle exec rails generate model Vote link:references user:references
% bundle exec rails db:migrate
```

これにより、Voteモデルが作成されます。
`app/models/vote.rb`
```ruby
class Vote < ApplicationRecord
  belongs_to :link, validate: true
  belongs_to :user, validate: true
end
```

次に、VoteTypeを追加します。
```bash
% bundle exec rails generate graphql:object VoteType id:ID! user:UserType! link:LinkType!
```

CreateVoteリゾルバを定義します。
```ruby
module Mutations
    class CreateVote < BaseMutation
      argument :link_id, ID, required: false
  
      type Types::VoteType
  
      def resolve(link_id: nil)
        Vote.create!(
          link: Link.find(link_id),
          user: context[:current_user]
        )
      end
    end
  end
```

最後に、create_voteフィールドをmutation_typeに追加します。
```ruby
module Types
  class MutationType < Types::BaseObject
    field :create_link, mutation: Mutations::CreateLink
    field :create_user, mutation: Mutations::CreateUser
    field :sign_in_user, mutation: Mutations::SignInUser
    field :create_vote, mutation: Mutations::CreateVote
  end
end

```

https://zenn.dev/tossy21/articles/d0bb29cb0fcd2b#%E3%82%B5%E3%82%A4%E3%83%B3%E3%82%A4%E3%83%B3%E3%81%99%E3%82%8Bmutation と同様に、SignInUser mutationを実行し、サインインしておきます。
その後、createVote mutationを実行してください。
http://127.0.0.1:3000/graphiql

```
mutation {
  createVote(input:{
    linkId: "9"
  }
    
  ){
    link {
      description
    }
    user{
      name
    }
  }
  
}
```

私の環境だと、`context[:current_user] `がうまくいかず、`app/graphql/mutations/create_vote.rb`を以下のように変更すると成功しました。

```ruby
...
      def resolve(link_id: nil)
        Vote.create!(
            link_id: link_id,
          user:  User.find_by(id: 1) 
        )
...
```

成功した際の応答は以下。
```
{
  "data": {
    "createVote": {
      "link": {
        "description": "Your testing playground"
      },
      "user": {
        "name": "Test User"
      }
    }
  }
}
```


## 投票とリンクを関連付ける

投票を作成できましたが、現時点では投票を取得する方法がありません。
各リンクの投票を取得できるようにするため、LinkモデルにVotesモデルを関連づけます。
`app/models/link.rb`
```ruby
class Link < ApplicationRecord
    belongs_to :user, optional: true

    has_many :votes
end
```

次に、LinkTypeにvotesフィールドを追加します。
```ruby
module Types
  class LinkType < Types::BaseObject
    field :id, ID, null: false
    field :url, String, null: false
    field :description, String, null: false
    field :posted_by, UserType, null: true, method: :user
    field :votes, [Types::VoteType], null: false
  end
end
```

これで、allLinksのQueryを叩くと、すべての投票を確認できます。

```
query{
  allLinks{
    votes{
      id
    }
  }
}
```

応答
```
{
  "data": {
    "allLinks": [
      {
        "votes": []
      },
...
      {
        "votes": []
      },
      {
        "votes": [
          {
            "id": "1"
          },
          {
            "id": "2"
          }
        ]
      }
    ]
  }
}
```

## ユーザと投票を関連付ける
次にユーザが行ったすべての投票を見つけられるようにします。
`app/models/user.rb`
```ruby
class User < ApplicationRecord
    has_secure_password

    validates :email, presence: true, uniqueness: true
    validates :password, presence: true

    has_many :votes
    has_many :links
end

```

次にUserTypeにlinksフィールドを追加します。
```ruby
module Types
    class UserType < Types::BaseObject
      field :id, ID, null: false
      field :created_at, GraphQL::Types::ISO8601DateTime, null: false
      field :name, String, null: false
      field :email, String, null: false
      field :votes, [Types::VoteType], null: false
      field :links, [Types::LinkType], null: false
    end
  end

```

ユーザの全ての投票を確認します。
```
query{
  allLinks{
    id
    postedBy{
      name
      votes{
        link{
          description
        }
      }
    }
  }
}
```

応答。
ここは、current_userに正しくセッション情報が入っていれば、うまく情報が取得できると思われます。
```
{
  "data": {
    "allLinks": [
      {
        "id": "1",
        "postedBy": null
      },
   ...
      {
        "id": "9",
        "postedBy": null
      }
    ]
  }
}
```
