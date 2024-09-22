---
title: "How to GraphQLを試す(Authentication)"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GraphQL", "Ruby"]
published: true
---

# 前記事
この記事は、以下の続きです。
https://zenn.dev/tossy21/articles/d6b61d27079e83


# 認証

本記事では、How to GraphQLの「認証」を試していきます。

https://www.howtographql.com/graphql-ruby/4-authentication/

## ユーザの作成

ユーザを作成するmutationを実装します。
最初に、Userモデルを作り、migrateしておきます。

```bash
% bundle exec rails g model User name email passwo
rd_digest
      invoke  active_record
      create    db/migrate/20240922121736_create_users.rb
      create    app/models/user.rb
      invoke    test_unit
      create      test/models/user_test.rb
      create      test/fixtures/users.yml

% % bundle exec rails db:migrate
== 20240922121736 CreateUsers: migrating ======================================
-- create_table(:users)
   -> 0.0026s
== 20240922121736 CreateUsers: migrated (0.0027s) =============================
```

`app/models/user.rb`を以下のように変更します。
```ruby
class User < ApplicationRecord
    has_secure_password

    validates :email, presence: true, uniqueness: true
    validates :password, presence: true
end
```

has_secure_passwordでは、ユーザのパスワードを暗号化して検証するためにbcrypt gem が必要です。
Gemfileに以下を追加します。
```gemfile
gem "bcrypt", "~> 3.1.13"
```

```bash
% bundle install
```

次に、ユーザを表すGraphQLタイプを作成します。
```bash
% bundle exec rails generate graphql:object UserType id:ID! name:String! email:String!
```

認証プロバイダーの機能を持つAuthProviderCredentialsInputタイプを作成します。
`app/graphql/types/auth_provider_credentials_input.rb`
```ruby
module Types
    class AuthProviderCredentialsInput < BaseInputObject
      # the name is usually inferred by class name but can be overwritten
      graphql_name 'AUTH_PROVIDER_CREDENTIALS'
  
      argument :email, String, required: true
      argument :password, String, required: true
    end
  end
```

次に`app/graphql/mutations/create_user.rb`にて、ユーザを作成するためのmutationを作成します。
```ruby
module Mutations
    class CreateUser < BaseMutation
      class AuthProviderSignupData < Types::BaseInputObject
        argument :credentials, Types::AuthProviderCredentialsInput, required: false
      end
  
      argument :name, String, required: true
      argument :auth_provider, AuthProviderSignupData, required: false
  
      type Types::UserType
  
      def resolve(name: nil, auth_provider: nil)
        User.create!(
          name: name,
          email: auth_provider&.[](:credentials)&.[](:email),
          password: auth_provider&.[](:credentials)&.[](:password)
        )
      rescue ActiveRecord::RecordInvalid => e
        GraphQL::ExecutionError.new("Invalid input: #{e.record.errors.full_messages.join(', ')}")
      end
    end
  end

```

上記をmutationリストに追加します。
```ruby
module Types
  class MutationType < Types::BaseObject
    field :create_link, mutation: Mutations::CreateLink
    field :create_user, mutation: Mutations::CreateUser
  end
end

```

`app/graphql/type/user_type.rb`を以下にします。
```ruby
module Types
    class UserType < Types::BaseObject
      field :id, ID, null: false
      field :name, String, null: false
      field :email, String, null: false
    end
  end

```

これで、GraphiQLを利用して、新規ユーザを作成します。
http://127.0.0.1:3000/graphiql

```
mutation {
  createUser(input: {
    name: "Test User",
    authProvider: {
      credentials: {
        email: "email@example.com",
        password: "123456"
      }
    }
  }) {
    id
    name
    email
  }
}
```

すると、以下の応答が返ってきます。
```
{
  "data": {
    "createUser": {
      "id": "1",
      "name": "Test User",
      "email": "email@example.com"
    }
  }
}
```

ユーザが作れているか、Rails consoleで確認します。
```bash
% bundle exec rails c
Loading development environment (Rails 7.2.1)
graphql-tutorial(dev)> User.all
  User Load (3.4ms)  SELECT "users".* FROM "users" /* loading for pp */ LIMIT 11 /*application='GraphqlTutorial'*/
=> 
[#<User:0x0000000109f267f0
  id: 1,
  name: "Test User",
  email: "[FILTERED]",
  password_digest: "[FILTERED]",
  created_at: "2024-09-22 13:23:52.493057000 +0000",
  updated_at: "2024-09-22 13:23:52.493057000 +0000">]
```

## サインインするmutation

ユーザがいる場合に、GraphQLを使用して、ユーザをサインインさせます。
mutationのリゾルバを作成します。
`app/graphql/mutations/sign_in_user.rb`
```ruby
module Mutations
  class SignInUser < BaseMutation
    null true

    argument :credentials, Types::AuthProviderCredentialsInput, required: false

    field :token, String, null: true
    field :user, Types::UserType, null: true

    def resolve(credentials: nil)
      # basic validation
      return unless credentials

      user = User.find_by email: credentials[:email]

      # ensures we have the correct user
      return unless user
      return unless user.authenticate(credentials[:password])

      # use Ruby on Rails - ActiveSupport::MessageEncryptor, to build a token
      crypt = ActiveSupport::MessageEncryptor.new(Rails.application.credentials.secret_key_base.byteslice(0..31))
      token = crypt.encrypt_and_sign("user-id:#{ user.id }")

      { user: user, token: token }
    end
  end
end
```

作成したmutationのリゾルバをmutation listに追加します。
`app/graphql/types/mutation_type.rb`

```ruby
module Types
  class MutationType < Types::BaseObject
    field :create_link, mutation: Mutations::CreateLink
    field :create_user, mutation: Mutations::CreateUser
    field :sign_in_user, mutation: Mutations::SignInUser
  end
end
```

これで、GraphiQLを利用して、トークンを取得できます。
http://127.0.0.1:3000/graphiql

```
mutation {
  signInUser(input: {
    credentials:{
        email: "email@example.com",
        password: "123456"
    }
  }) {
	token
    user{
      id
    }
  }
}
```

応答は以下。
```
{
  "data": {
    "signInUser": {
      "token": "xxx",
      "user": {
        "id": "1"
      }
    }
  }
}
```

## リクエストの認証
SignInUser mutationにより提供されるトークンを使用して、アプリは後続のリクエストを認証できます。

`app/graphql/graphql_controller.rb`
```ruby
class GraphqlController < ApplicationController
    def execute
      variables = ensure_hash(params[:variables])
      query = params[:query]
      operation_name = params[:operationName]
      context = {
        # we need to provide session and current user
        session: session,
        current_user: current_user
      }
      result = GraphqlTutorialSchema.execute(query, variables: variables, context: context, operation_name: operation_name)
      render json: result
    rescue => e
      raise e unless Rails.env.development?
      handle_error_in_development e
    end
  
    private
  
    # gets current user from token stored in the session
    def current_user
      # if we want to change the sign-in strategy, this is the place to do it
      return unless session[:token]
  
      crypt = ActiveSupport::MessageEncryptor.new(Rails.application.credentials.secret_key_base.byteslice(0..31))
      token = crypt.decrypt_and_verify session[:token]
      user_id = token.gsub('user-id:', '').to_i
      User.find user_id
    rescue ActiveSupport::MessageVerifier::InvalidSignature
      nil
    end
  
    # Handle form data, JSON body, or a blank value
    def ensure_hash(ambiguous_param)
      # ...code
    end
  
    def handle_error_in_development(e)
      # ...code
    end
  end

```

SignInUserのmutationリゾルバを修正します。
`app/graphql/mutations/sign_in_user.rb`

```ruby
        crypt = ActiveSupport::MessageEncryptor.new(Rails.application.credentials.secret_key_base.byteslice(0..31))
        token = crypt.encrypt_and_sign("user-id:#{ user.id }")

        context[:session][:token] = token
```

## 作成されたリンクにユーザーをリンクする

データベースのmigrationファイルを作成します。

```bash
% bundle exec rails generate migration add_user_id_link
```

作成されたmigrationファイルを以下に変更します。
```ruby
class AddUserIdLink < ActiveRecord::Migration[7.2]
  def change
    change_table :links do |t|
      t.references :user, foreign_key: true
    end
  end
end

```

次に、migrationを行います。
```bash
% bundle exec rails db:migrate
```

次に、usersテーブルとの関連付けを行います。
Linkはuserにbelongs_toで紐づきます。
`app/models/link.rb`
```ruby
class Link < ApplicationRecord
    belongs_to :user
end
```

また、LinkTypeを以下に更新します。
`app/graphql/types/link_type.rb`
```ruby
# frozen_string_literal: true

module Types
  class LinkType < Types::BaseObject
    field :id, ID, null: false
    field :url, String, null: false
    field :description, String, null: false
    field :posted_by, UserType, null: true, method: :user
  end
end
```

リゾルバを更新します。
`app/graphql/mutations/create_link.rb`
```ruby
module Mutations
    class CreateLink < BaseMutation
      # arguments passed to the `resolve` method
      argument :description, String, required: true
      argument :url, String, required: true

      # return type from the mutation
      type Types::LinkType

      def resolve(description: nil, url: nil)
        Link.create!(
         description: description,
          url: url,
        user: context[:current_user]
        )
      end
    end
  end
```

`app/controllers/graphql_controller.rb`を修正します。
```ruby
# frozen_string_literal: true

class GraphqlController < ApplicationController
  # If accessing from outside this domain, nullify the session
  # This allows for outside API access while preventing CSRF attacks,
  # but you'll have to authenticate your user separately
  # protect_from_forgery with: :null_session

  def execute
    variables = prepare_variables(params[:variables])
    query = params[:query]
    operation_name = params[:operationName]
    context = {
      # Query context goes here, for example:
      # current_user: current_user,
      session: session,
      current_user: current_user
    }
    result = GraphqlTutorialSchema.execute(query, variables: variables, context: context, operation_name: operation_name)
    render json: result
  rescue StandardError => e
    raise e unless Rails.env.development?
    handle_error_in_development(e)
  end

  private

  # Handle variables in form data, JSON body, or a blank value
  def prepare_variables(variables_param)
    case variables_param
    when String
      if variables_param.present?
        JSON.parse(variables_param) || {}
      else
        {}
      end
    when Hash
      variables_param
    when ActionController::Parameters
      variables_param.to_unsafe_hash # GraphQL-Ruby will validate name and type of incoming variables.
    when nil
      {}
    else
      raise ArgumentError, "Unexpected parameter: #{variables_param}"
    end
  end

  def handle_error_in_development(e)
    logger.error e.message
    logger.error e.backtrace.join("\n")

    render json: { errors: [{ message: e.message, backtrace: e.backtrace }], data: {} }, status: 500
  end
end

```

`app/controllers/application_controller.rb`
```ruby
class ApplicationController < ActionController::API
  # Only allow modern browsers supporting webp images, web push, badges, import maps, CSS nesting, and CSS :has.
  #allow_browser versions: :modern

  # Assuming you have a method to find the current user
  def current_user
    # Logic to find the current user, e.g., from session or token
    @current_user ||= User.find_by(id: session[:user_id]) # 例
  end
end

```

`app/models/auth_token.rb`
```ruby
module AuthToken
    module_function
  
    PREFIX = 'user-id'.freeze
  
    def token_for_user(user)
      crypt.encrypt_and_sign("#{PREFIX}#{user.id}")
    end
  
    def user_from_token(token)
      return if token.blank?
  
      user_id = crypt.decrypt_and_verify(token).gsub(PREFIX, '').to_i
      User.find_by id: user_id
    rescue ActiveSupport::MessageVerifier::InvalidSignature
      nil
    end
  
    def crypt
      ActiveSupport::MessageEncryptor.new(
        Rails.application.credentials.secret_key_base.byteslice(0..31)
      )
    end
  end

```


では、 http://127.0.0.1:3000/graphiql で試します。
まずは、SignInUserを試します。
```
mutation {
  signInUser(input: {
    credentials:{
        email: "email@example.com",
        password: "123456"
    }
  }) {
	token
    user{
      id
    }
  }
}
```

応答
```
{
  "data": {
    "signInUser": {
      "token": "xxx",
      "user": {
        "id": "1"
      }
    }
  }
}
```

次に、createLink mutationを実行します。

```
mutation {
  createLink(input: {
   url: "http://127.0.0.1:3000/graphiql",
    description: "Your testing playground",
  }) {
    id
    url
    description
    postedBy{
      id
      name
    }

  }
}
```

うまくいくと、以下となります。
```
{
  "data": {
    "createLink": {
      "id": "1",
      "url": "http://localhost:3000/graphiql",
      "description": "Your testing playground",
      "postedBy": {
        "id": "1",
        "name": "Test User""
      }
    }
  }
}
```
