## 4. 意味のあるテストデータの作成

```
ここまで私たちは ごく普通の Ruby オブジェクト (一般的には plain old Ruby objects の略語 で PORO と呼ばれます)を使ってテスト用の一時データを作ってきました。
この方法はシンプルですし、余計な gem を追加する必要もありません。
もしあなたのアプリケーションにお いてテストデータを作るのにこの方法で事足りるのであれば、わざわざ余計なものを追加してテストスイートを複雑にする必要はありません。
ですが、テストシナリオが複雑になってもテストデータのセットアップはシンプルな方がいいですよね。
複雑なテストシナリオになったときでも、私たちはデータよりもテストに フォーカスしたいと思うはずです。
幸いなことに、テストデータを簡単にしてくれる Ruby ライブラリがいくつかあります。
この章では有名な gem である Factory Bot に焦点を当てま す。具体的には次のような内容を説明します。

• 他の方法と比較した場合のファクトリの利点と欠点について説明します。

• それから基本的なファクトリを作り、既存のスペックで使ってみます。

• 続いてファクトリを編集し、さらに便利で使いやすくします。

• さらに Active Record の関連を再現する、より高度なファクトリを見ていきます。

• 最後に、ファクトリを使いすぎるリスクについて説明します。
```

## ファクトリ対フィクスチャ

```
Rails ではサンプルデータを生成する手段として、フィクスチャと呼ばれる機能がデフォルトで提供されています。
フィクスチャは YAML 形式のファイルです。このファイルを使って サンプルデータを作成します。たとえば、Project モデルのフィクスチャなら次のようになります。
```

+ `project.yml`(作成しなくてよい 例である)<br>

```yml:project.yml
death_star:
  name: "Death Star"
  description: "Create the universe's ultimate battle station"
  due_on: 2016-08-29

rogue_one:
  name: "Steal Death Star plans"
  description: "Destroy the Empire's new battle station"
  due_on: 2016-08-29
```

```
それからテストの中で projects(:rogue_one) と呼び出すだけで、全属性がセットされた 新しい Project が使えるようになります。とても素晴らしいですね。
フィクスチャを好む人もたくさんいます。フィクスチャは比較的速いですし、Rails に最初 から付いてきます。あなたのアプリケーションやテストスイートに余計なものを追加する必 要がありません。
とはいえ、私が初心者だった頃はフィクスチャで困ったことを覚えています。私は実行中 のテストでどんなデータが作成されたのかを見たかったのですが、フィクスチャを使うとテ ストとは別のフィクスチャファイルに保存された値を覚えておく必要がありました。実際、 私は今でもテストデータはすぐに確認できる方が好きです。テストデータのセットアップは テストの一部として目に見える方がいいですし、テストが実行されたときに何が起きるのかも理解しやすくなります。
フィクスチャにはもろくて壊れやすいという性質もあります。これはすなわち、テストコ ードやアプリケーションコードを書くのと同じぐらいの時間をテストデータファイルの保守 に時間をかけなくてはならない、ということを意味しています。
最後に、Rails はフィクスチ ャのデータをデータベースに読み込む際に Active Record を使いません。これはどういうことでしょうか?これはつまり、モデルのバリデーションのような重要な機能が無視されるということです。
私に言わせれば、これは望ましくありません。なぜなら、本番環境のデータと一致しなくなる可能性があるからです。もし同じデータをWebフォームやコンソールから作ろうとすると、失敗することもあるわけです。これは困りますよね?
このような理由からテストデータのセットアップが複雑になってきたときは、私はファクトリを使っています。
ファクトリはシンプルで柔軟性に富んだテストデータ構築用のブロックです。
もし私がテストを理解するのに役立ったコンポーネントを一つだけ挙げなければならないとしたら、Factory Bot26を挙げると思います。最初は少しトリッキーで理解しにくいかもしれませんが、実際に使って一度基本を覚えれば比較的シンプルに使うことができます。 ファクトリは適切に(つまり賢明に)使えば、あなたのテストをきれいで読みやすく、リ
アルなものに保ってくれます。しかし、多用しすぎると遅いテストの原因になります(と、[お利口で声の大きい Rubyist たちが言っています27](https://accounts.google.com/v3/signin/identifier?dsh=S-1744182769%3A1663734475754521&continue=https%3A%2F%2Fgroups.google.com%2Fmy-groups&followup=https%3A%2F%2Fgroups.google.com%2Fmy-groups&osid=1&passive=1209600&flowName=GlifWebSignIn&flowEntry=ServiceLogin&ifkv=AQDHYWoCsASpUWnLHm09Ds1Pt-EbYIEnmTau48gxWXffFWDv4bQkmrxoviK3W8xJ5u7WSWg-Vhs8rA))。
彼らが言うことはわかりますし、気軽にファクトリを使うとスピードの面で高コストになる、というのも理解できます。
ですが、 それでも私は、遅いテストは何もテストがない状態よりもずっと良いと信じています。
特に 初心者にとっては絶対そうだと思います。
あなたがテストスイートを作りあげ、テストに慣 れてきたと思えるようになれば、あなたはいつでもファクトリをもっと効率の良いアプロー チに置き換えることができます。
```

## Factory Botをインストールする

+ `Gemfile`を編集<br>

```:Gemfile
source "https://rubygems.org"
git_source(:github) { |repo| "https://github.com/#{repo}.git" }

# Bundle edge Rails instead: gem "rails", github: "rails/rails", branch: "main"
gem "rails", "7.0.2.3"

# The original asset pipeline for Rails [https://github.com/rails/sprockets-rails]
gem "sprockets-rails"

# Use sqlite3 as the database for Active Record
gem "sqlite3", "~> 1.4"

# Use the Puma web server [https://github.com/puma/puma]
gem "puma", "~> 5.0"

# Use JavaScript with ESM import maps [https://github.com/rails/importmap-rails]
gem 'importmap-rails'

# Hotwire's SPA-like page accelerator [https://turbo.hotwired.dev]
gem "turbo-rails"

# Hotwire's modest JavaScript framework [https://stimulus.hotwired.dev]
gem "stimulus-rails"

# Build JSON APIs with ease [https://github.com/rails/jbuilder]
gem "jbuilder"

# Use Redis adapter to run Action Cable in production
# gem "redis", "~> 4.0"

# Use Kredis to get higher-level data types in Redis [https://github.com/rails/kredis]
# gem "kredis"

# Use Active Model has_secure_password [https://guides.rubyonrails.org/active_model_basics.html#securepassword]
# gem "bcrypt", "~> 3.1.7"

# Windows does not include zoneinfo files, so bundle the tzinfo-data gem
gem "tzinfo-data", platforms: %i[ mingw mswin x64_mingw jruby ]

# Reduces boot times through caching; required in config/boot.rb
gem "bootsnap", require: false

# Use Sass to process CSS
gem "sassc-rails"

# Use Active Storage variants [https://guides.rubyonrails.org/active_storage_overview.html#transforming-images]
gem "image_processing", "~> 1.2"

group :development, :test do
  # See https://guides.rubyonrails.org/debugging_rails_applications.html#debugging-with-the-debug-gem
  gem "debug", platforms: %i[ mri mingw x64_mingw ], require: false

  gem 'rspec-rails' # add in chapter 2
  gem 'factory_bot_rails' ### 追加
end

group :development do
  # Use console on exceptions pages [https://github.com/rails/web-console]
  gem "web-console"

  # Add speed badges [https://github.com/MiniProfiler/rack-mini-profiler]
  # gem "rack-mini-profiler"

  # Speed up commands on slow machines / big apps [https://github.com/rails/spring]
  # gem "spring"

  gem 'faker', require: false # for sample data in development
end

group :test do
  # Use system testing [https://guides.rubyonrails.org/testing.html#system-testing]
  # TODO: waiting for release https://github.com/teamcapybara/capybara/pull/2520
  gem "capybara", github: 'teamcapybara/capybara'
  gem "selenium-webdriver"
  gem "webdrivers"

  gem 'launchy'
  gem 'shoulda-matchers'
  gem 'vcr'
  gem 'webmock'
end

gem 'devise'
gem 'net-imap'
gem 'net-pop'
gem 'net-smtp'
gem 'activestorage-validator'
gem 'geocoder'
```

+ `bundle install`を実行<br>

+ `config/application.rb`を編集<br>

```rb:application.rb
require_relative "boot"

require "rails/all"

# Require the gems listed in Gemfile, including any gems
# you've limited to :test, :development, or :production.
Bundler.require(*Rails.groups)

module Projects
  class Application < Rails::Application
    # Initialize configuration defaults for originally generated Rails version.
    config.load_defaults 7.0

    # Configuration for the application, engines, and railties goes here.
    #
    # These settings can be overridden in specific environments using the files
    # in config/environments, which are processed later.
    #
    # config.time_zone = "Central Time (US & Canada)"
    # config.eager_load_paths << Rails.root.join("extras")

    # 編集
    config.generators do |g|
      g.test_framework :rspec,
        view_specs: false,
        helper_specs: false,
        routing_specs: false
    end
    # ここまで
  end
end
```

## アプリケーションにファクトリを追加する

+ `$ bin/rails g factory_bot:model user`を実行<br>

+ `spec/factories/users.rb`を編集<br>

```rb:users.rb
FactoryBot.define do
  factory :user do
    first_name { "Aaron" } # FacrotyBot4.11から中括弧が必要になった
    last_name { "Summer" }
    email { "tester@example.com" }
    password { "dottle-nouveau-pavilion-tights-furze" }
  end
end
```

+ `spec/models/user_spec.rb`を編集<br>

```rb:user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do
  # 追加
  # 有効なファクトリを持つこと
  it "has a valid factory" do
    expect(FactoryBot.build(:user)).to be_valid
  end
  # ここまで

  # 姓、名、メール、パスワードがあれば有効な状態であること
  it "is valid with a first name, last name, email, and password" do
    user = User.new(
      first_name: "Aaron",
      last_name: "Summer",
      email: "tester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )
    expect(user).to be_valid
  end

  # 名がなければ無効な状態であること
  it "is invalid without a first name" do
    user = User.new(first_name: nil)
    user.valid?
    expect(user.errors[:first_name]).to include("can't be blank")
  end

  # 姓がなければ無効な状態であること
  it "is invalid without a last name" do
    user = User.new(last_name: nil)
    user.valid?
    expect(user.errors[:last_name]).to include("can't be blank")
  end

  # メールアドレスがなければ無効な状態であること
  it "is invalid without an email address" do
    user = User.new(email: nil)
    user.valid?
    expect(user.errors[:email]).to include("can't be blank")
  end

  # 重複したアドレスなら無効な状態であること
  it "is invalid with a duplicate email address" do
    User.create(
      first_name: "Joe",
      last_name: "Tester",
      email: "tester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )
    user = User.new(
      first_name: "Jane",
      last_name: "Tester",
      email: "tester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze"
    )
    user.valid?
    expect(user.errors[:email]).to include("has already been taken")
  end

  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string" do
    user = User.new(
      first_name: 'John',
      last_name: "Doe",
      email: "johndoe@example.com",
    )
    expect(user.name).to eq "John Doe"
  end
end
```

+ `spec/models/user_spec.rb`を編集<br>

```rb:user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do
  # 有効なファクトリを持つこと
  it "has a valid factory" do
    expect(FactoryBot.build(:user)).to be_valid
  end

  # 姓、名、メール、パスワードがあれば有効な状態であること
  it "is valid with a first name, last name, email, and password" do
    user = User.new(
      first_name: "Aaron",
      last_name: "Summer",
      email: "tester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )
    expect(user).to be_valid
  end

  # 編集
  # 名がなければ無効な状態であること
  it "is invalid without a first name" do
    user = FactoryBot.build(:user, first_name: nil)
    user.valid?
    expect(user.errors[:first_name]).to include("can't be blank")
  end

  # 姓がなければ無効な状態であること
  it "is invalid without a last name" do
    user = FactoryBot.build(:user, last_name: nil)
    user.valid?
    expect(user.errors[:last_name]).to include("can't be blank")
  end

  # メールアドレスがなければ無効な状態であること
  it "is invalid without an email address" do
    user = FactoryBot.build(:user, email: nil)
    user.valid?
    expect(user.errors[:email]).to include("can't be blank")
  end
  # ここまで

  # 重複したアドレスなら無効な状態であること
  it "is invalid with a duplicate email address" do
    User.create(
      first_name: "Joe",
      last_name: "Tester",
      email: "tester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )
    user = User.new(
      first_name: "Jane",
      last_name: "Tester",
      email: "tester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze"
    )
    user.valid?
    expect(user.errors[:email]).to include("has already been taken")
  end

  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string" do
    user = User.new(
      first_name: 'John',
      last_name: "Doe",
      email: "johndoe@example.com",
    )
    expect(user.name).to eq "John Doe"
  end
end
```

+ `$ bundle exec rspec`を実行<br>

+ `spec/models/user_spec.rb`を編集<br>

```rb:user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do
  # 有効なファクトリを持つこと
  it "has a valid factory" do
    expect(FactoryBot.build(:user)).to be_valid
  end

  # 姓、名、メール、パスワードがあれば有効な状態であること
  it "is valid with a first name, last name, email, and password" do
    user = User.new(
      first_name: "Aaron",
      last_name: "Summer",
      email: "tester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )
    expect(user).to be_valid
  end

  # 名がなければ無効な状態であること
  it "is invalid without a first name" do
    user = FactoryBot.build(:user, first_name: nil)
    user.valid?
    expect(user.errors[:first_name]).to include("can't be blank")
  end

  # 姓がなければ無効な状態であること
  it "is invalid without a last name" do
    user = FactoryBot.build(:user, last_name: nil)
    user.valid?
    expect(user.errors[:last_name]).to include("can't be blank")
  end

  # メールアドレスがなければ無効な状態であること
  it "is invalid without an email address" do
    user = FactoryBot.build(:user, email: nil)
    user.valid?
    expect(user.errors[:email]).to include("can't be blank")
  end

  # 重複したアドレスなら無効な状態であること
  it "is invalid with a duplicate email address" do
    User.create(
      first_name: "Joe",
      last_name: "Tester",
      email: "tester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )
    user = User.new(
      first_name: "Jane",
      last_name: "Tester",
      email: "tester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze"
    )
    user.valid?
    expect(user.errors[:email]).to include("has already been taken")
  end

  # 編集
  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string" do
    user = FactoryBot.build(:user, first_name: "John", last_name: "Doe")
    expect(user.name).to eq "John Doe"
  end
  # ここまで
end
```

+ `spec/models/user_spec.rb`を編集<br>

```rb:user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do
  # 有効なファクトリを持つこと
  it "has a valid factory" do
    expect(FactoryBot.build(:user)).to be_valid
  end

  # 姓、名、メール、パスワードがあれば有効な状態であること
  it "is valid with a first name, last name, email, and password" do
    user = User.new(
      first_name: "Aaron",
      last_name: "Summer",
      email: "tester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )
    expect(user).to be_valid
  end

  # 名がなければ無効な状態であること
  it "is invalid without a first name" do
    user = FactoryBot.build(:user, first_name: nil)
    user.valid?
    expect(user.errors[:first_name]).to include("can't be blank")
  end

  # 姓がなければ無効な状態であること
  it "is invalid without a last name" do
    user = FactoryBot.build(:user, last_name: nil)
    user.valid?
    expect(user.errors[:last_name]).to include("can't be blank")
  end

  # メールアドレスがなければ無効な状態であること
  it "is invalid without an email address" do
    user = FactoryBot.build(:user, email: nil)
    user.valid?
    expect(user.errors[:email]).to include("can't be blank")
  end

  # 編集
  # 重複したアドレスなら無効な状態であること
  it "is invalid with a duplicate email address" do
    FactoryBot.create(:user, email: "aaron@example.com")
    user = FactoryBot.build(:user, email: "aaron@example.com")
    user.valid?
    expect(user.errors[:email]).to include("has already been taken")
  end
  # ここまで

  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string" do
    user = FactoryBot.build(:user, first_name: "John", last_name: "Doe")
    expect(user.name).to eq "John Doe"
  end
end
```

```
この example ではテストオブジェクトの email 属性が重複しないことを確認しています。
これを検証するためには二つ目(訳注: 原文は「二つ目」になっていますが、「一つ目」が正 だと思われます)の User がデータベースに保存されている必要があります。
そこでエクスペクテーションを実行する前に、FactoryBot.create を使って同じメールアドレスの user を最初に保存しているのです。

※ 次のことを覚えてください。FactoryBot.build を使うと新しいテストオブジェクト をメモリ内に保存します。
FactoryBot.create を使うとアプリケーションのテスト用データベースにオブジェクトを永続化します。
```

## シーケンスを使ってユニークなデータを生成する

```
現在の User ファクトリにはちょっと問題があります。
もし、先ほどの example を次のように書いていたら何が起きると思いますか?
```

+ `spec/models/user_spec.rb`を編集(下記コードでもパスしてしまう)<br>

```rb:user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do
  # 有効なファクトリを持つこと
  it "has a valid factory" do
    expect(FactoryBot.build(:user)).to be_valid
  end

  # 姓、名、メール、パスワードがあれば有効な状態であること
  it "is valid with a first name, last name, email, and password" do
    user = User.new(
      first_name: "Aaron",
      last_name: "Summer",
      email: "tester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )
    expect(user).to be_valid
  end

  # 名がなければ無効な状態であること
  it "is invalid without a first name" do
    user = FactoryBot.build(:user, first_name: nil)
    user.valid?
    expect(user.errors[:first_name]).to include("can't be blank")
  end

  # 姓がなければ無効な状態であること
  it "is invalid without a last name" do
    user = FactoryBot.build(:user, last_name: nil)
    user.valid?
    expect(user.errors[:last_name]).to include("can't be blank")
  end

  # メールアドレスがなければ無効な状態であること
  it "is invalid without an email address" do
    user = FactoryBot.build(:user, email: nil)
    user.valid?
    expect(user.errors[:email]).to include("can't be blank")
  end

  # 編集
  # 重複したアドレスなら無効な状態であること
  it "is invalid with a duplicate email address" do
    FactoryBot.create(:user)
    user = FactoryBot.build(:user)
    user.valid?
    expect(user.errors[:email]).to include("has already been taken")
  end
  # ここまで

  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string" do
    user = FactoryBot.build(:user, first_name: "John", last_name: "Doe")
    expect(user.name).to eq "John Doe"
  end
end
```

```
うーん、このテストもやっぱりパスしますね。
なぜなら、明示的に異なる値を設定しない 限り、ファクトリが常にユーザーのメールアドレスを tester@example.com に設定するからです。
これはここまでに書いてきたスペックでは問題になっていませんが、ファクトリで複数のユーザーをセットアップする必要が出てきた場合は実際のテストコードが走る前に例外が 発生します。たとえば以下のような場合です。
```

+ `Sample`<br>

```rb:example_spec.rb
# 複数のユーザーで何かする
it "does something with multiple users" do
user1 = FactoryBot.create(:user)
user2 = FactoryBot.create(:user)
expect(true).to be_truthy
end
```

+ すると次のようなバリデーションエラーが発生します。<br>

```:terminal
Failures:
1) User does something with multiple users Failure/Error: user2 = FactoryBot.create(:user)
     ActiveRecord::RecordInvalid:
       Validation failed: Email has already been taken
```

```
Factory Bot では シーケンス を使ってこのようなユニークバリデーションを持つフィールドを扱うことができます。
シーケンスはファクトリから新しいオブジェクトを作成するたびに、カウンタの値を1つずつ増やしながら、ユニークにならなければいけない属性に値を設定します。
ファクトリ内にシーケンスを作成して実際に使ってみましょう。
```

+ `spec/factories/user.rb`を編集<br>

```rb:user.rb
FactoryBot.define do
  factory :user do
    first_name { "Aaron" }
    last_name { "Summer" }
    sequence(:email) { |n| "tester#{n}@example.com" } # 編集
    password { "dottle-nouveau-pavilion-tights-furze" }
  end
end
```

```
メール文字列に n の値がどのように挟み込まれるかわかりますか?
こうすれば新しいユー ザーを作成するたびに、
tester1@example.com 、tester2@example.com というように、ユニークで連続したメールアドレスが設定されます。
```

+ `spec/models/user_spec.rb`を編集<br>

```rb:user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do
  # 有効なファクトリを持つこと
  it "has a valid factory" do
    expect(FactoryBot.build(:user)).to be_valid
  end

  # 姓、名、メール、パスワードがあれば有効な状態であること
  it "is valid with a first name, last name, email, and password" do
    user = User.new(
      first_name: "Aaron",
      last_name: "Summer",
      email: "tester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )
    expect(user).to be_valid
  end

  # 名がなければ無効な状態であること
  it "is invalid without a first name" do
    user = FactoryBot.build(:user, first_name: nil)
    user.valid?
    expect(user.errors[:first_name]).to include("can't be blank")
  end

  # 姓がなければ無効な状態であること
  it "is invalid without a last name" do
    user = FactoryBot.build(:user, last_name: nil)
    user.valid?
    expect(user.errors[:last_name]).to include("can't be blank")
  end

  # メールアドレスがなければ無効な状態であること
  it "is invalid without an email address" do
    user = FactoryBot.build(:user, email: nil)
    user.valid?
    expect(user.errors[:email]).to include("can't be blank")
  end

  # 戻す
  # 重複したアドレスなら無効な状態であること
  it "is invalid with a duplicate email address" do
    FactoryBot.create(:user, email: "aaron@example.com")
    user = FactoryBot.build(:user, email: "aaron@example.com")
    user.valid?
    expect(user.errors[:email]).to include("has already been taken")
  end
  # ここまで

  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string" do
    user = FactoryBot.build(:user, first_name: "John", last_name: "Doe")
    expect(user.name).to eq "John Doe"
  end
end
```

## ファクトリで関連を扱う

+ `$ bin/rails g factory_bot:model note`を実行<br>

+ `spec/factorires/notes.rb`を編集<br>

```rb:notes.rb
FactoryBot.define do
  factory :note do
    message { "My important note." }
    association :project
    association :user
  end
end
```

+ `$ bin/rails g factory_bot:model project`を実行<br>

+ `spec/factories/projects.rb`を編集<br>

```rb:projects.rb
FactoryBot.define do
  factory :project do
    sequence(:name) { |n| "Project #{n}" }
    description { "A test project." }
    due_on { 1.week.from_now }
    association :owner
  end
end
```

```
先へ進む前に、ユーザーファクトリにちょっとだけ情報を追加しましょう。
ファクトリに 名前を付けている2行目に、以下に示すような owner という別名(alias)を付けてください。
なぜこうする必要があるのかはすぐにわかります。
```

+ `spec/factories/users.rb`を編集<br>

```rb:users.rb
FactoryBot.define do
  factory :user, aliases: [:owner] do # 編集
    first_name { "Aaron" }
    last_name { "Summer" }
    sequence(:email) { |n| "tester#{n}@example.com" }
    password { "dottle-nouveau-pavilion-tights-furze" }
  end
end
```

```
メモはプロジェクトとユーザーの両方に属しています。
しかし、テストのたびにいちいち 手作業でプロジェクトとユーザーを作りたくありません。
私たちが作りたいのはメモだけで す。こちらの使用例を見てください。
```

+ `spec/models/note_spec.rb`を編集<br>

```rb:note_spec.rb
require 'rails_helper'

RSpec.describe Note, type: :model do
  # 追加
  # ファクトリで関連するデータを作成する
  it "generates associated data from a factory" do
    note = FactoryBot.create(:note)
    puts "This note's project is #{note.project.inspect}"
    puts "This note's user is #{note.user.inspect}"
  end
  # ここまで

  before do
    # このファイルの全てストで使用するテストデータをセットアップする
    @user = User.create(
      first_name: "Joe",
      last_name: "Tester",
      email:     "joetester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )

    @project = @user.projects.create(
      name: "Test Project",
    )
  end

  # ユーザー、プロジェクト、メッセージがあれば有効な状態であること
  it "is valid a user, project, and message" do
    note = Note.new(
      message: "This is a same note.",
      user: @user,
      project: @project
    )
    expect(note).to be_valid
  end

  # メッセージがなければ無効な状態であること
  it "is invalid without a message" do
    note = Note.new(message: nil)
    note.valid?
    expect(note.errors[:message]).to include("can't be blank")
  end

  # バリデーションのテストが並ぶ

  # 文字列に一致するメッセージを検索する
  describe "search message for a term" do

    before do
      # 検索機能の全テストに関連する追加のテストデータをセットアップする
      @note1 = @project.notes.create(
        message: "This is the first note.",
        user: @user
      )
      @note2 = @project.notes.create(
        message: "This is the second note.",
        user: @user,
      )
      @note3 = @project.notes.create(
        message: "First, preheat the oven.",
        user: @user,
      )
    end

    # 一致するデータが見つかるとき
    context "when a match is found" do
      # 一致する場合の examle が並ぶ ...
      # 文字列に一致するメモを返すこと
      it "returns notes that match the search term" do
        expect(Note.search("first")).to include(@note1, @note3)
      end
    end

    # 一致するデータが1件も見つからないとき
    context "when no match is found" do
      # 一致しない場合の example が並ぶ ...
      # 空のコレクションを返すこと
      it "returns an empty collection" do
        expect(Note.search("message")).to be_empty
      end
    end
  end
end
```

```
ここでは Factory Bot を1回しか呼んでいないにもかかわらず、テストの実行結果を見ると 必要なデータが全部作成されています。
```

```:terminal
Note
This note's project is #<Project id: 2, name: "Project 1", description: "A test project.", due_on: "2022-09-30", created_at: "2022-09-23 07:39:44.670856000 +0000", updated_at: "2022-09-23 07:39:44.670856000 +0000", user_id: 2>
This note's user is #<User id: 3, email: "tester2@example.com", created_at: "2022-09-23 07:39:44.675943000 +0000", updated_at: "2022-09-23 07:39:44.675943000 +0000", first_name: "Aaron", last_name: "Summer", authentication_token: [FILTERED], location: nil>
```

```
ですが、この例はファクトリで関連を扱う際の潜在的な落とし穴も示しています。
みなさん はわかりますか?ユーザーのメールアドレスをよく見てください。
なぜ tester1@example.com ではなく、tester2@example.com になっているのでしょうか?
この理由はメモのファクトリが 関連するプロジェクトを作成する際に関連するユーザー(プロジェクトに関連する owner) を作成し、それから2番目のユーザー(メモに関連するユーザー)を作成するからです。
この問題を回避するためにメモのファクトリを次のように更新します。こうするとデフォ ルトでユーザーが1人しか作成されなくなります。
```

+ `spec/factories/notes.rb`を編集<br>

```rb:notes.rb
FactoryBot.define do
  factory :note do
    message { "My important note." }
    association :project
    user { project.owner } # 編集
  end
end
```

+ `$ bundle exec rspec`を実行<br>

```
スペックの結果を見てもユーザーは1人だけです。

Note
This note's project is #<Project id: 2, name: "Project 1", description: "A test project.", due_on: "2022-09-30", created_at: "2022-09-23 07:46:24.068014000 +0000", updated_at: "2022-09-23 07:46:24.068014000 +0000", user_id: 2>
This note's user is #<User id: 2, email: "tester1@example.com", created_at: "2022-09-23 07:46:24.063358000 +0000", updated_at: "2022-09-23 07:46:24.063358000 +0000", first_name: "Aaron", last_name: "Summer", authentication_token: [FILTERED], location: nil>

ここで確認してほしいのは、ファクトリを使うとたまにびっくりするような結果が生まれるということです。
また、あなたの予想よりも多いテストデータが作成されることもありま す。このセクションで使ったようなちょっと不自然なテストコードであれば大した問題になりませんが、ユーザーの件数を検証するようなテストを書いているときはデバッグするときに原因を突き止めるのに苦労するかもしれません。
さて、ちょっと前に User ファクトリに追加した alias に戻りましょう。Project モデルを見ると、User の関連は owner という名前になっているのがわかると思います。
```

+ `app/models/project.rb`<br>

```rb:project.rb
class Project < ApplicationRecord
  validates :name, presence: true, uniqueness: { scope: :user_id }

  belongs_to :owner, class_name: 'User', foreign_key: :user_id # Userの関連は ownerになっている
  has_many :notes, dependent: :destroy
  has_many :tasks, dependent: :destroy

  attribute :due_on, :date, default: -> { Date.current }

  def late?
    due_on.in_time_zone < Date.current.in_time_zone
  end
end
```

```
このように Factory Bot を使う際はユーザーファクトリに対して owner という名前で参照される場合があると伝えなくてはいけません。そのために使うのが alias です。
alias の付いたユ ーザーファクトリのコード全体を再度載せておきます。
```

+ `spec/factories/users.rb`(再掲)<br>

```rb:users.rb
FactoryBot.define do
  factory :user, aliases: [:owner] do
    first_name { "Aaron" }
    last_name { "Summer" }
    sequence(:email) { |n| "tester#{n}@example.com" }
    password { "dottle-nouveau-pavilion-tights-furze" }
  end
end
```

+ 次へ続く<br>
