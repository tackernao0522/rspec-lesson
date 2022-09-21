## 02 Setup

+ [サンプルソースコード](https://github.com/JunichiIto/everydayrails-rspec-jp-2022) <br>

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

  gem 'rspec-rails' # 追加 add in chapter 2
  gem 'factory_bot_rails'
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

+ `$ bundle install`を実行<br>

+ `$ bin/setup`を実行(データベースのセットアップ等)<br>

+ `$ gem install foreman`を実行<br>

+ `$ rails server`を実行<br>

+ `$bin/rails g rspec:install`を実行<br>

+ `.rspec`を編集<br>

```:.rspec
--require spec_helper
--format documentation
```

## Rspecを試してみる

+ `$ bundle exec rspec`を実行 (下記のようになればOK)<br>

```
No examples found.

Finished in 0.00118 seconds (files took 0.1926 seconds to load)
0 examples, 0 failures
```

## rspec `binstub`を使って短いコマンドを実行できるようにする

```
Rails アプリケーションは Bundler の使用が必須であるため、RSpec を実行する際は bundle exec rspec のように毎回 bundle exec を付ける必要があります。binstub を作成すると bin/rspec のように少しタイプ量を減らすことができます。
```

+ `$ bundle binstubs rspec-core`を実行<br>

## ジェネレータ

```
さらに、もうひとつ手順があります。rails generate コマンドを使ってアプリケーション にコードを追加する際に、RSpec 用のスペックファイルも一緒に作ってもらうよう Rails を設 定しましょう。
RSpec はもうインストール済みなので、Rails のジェネレータを使っても、もともとデフォ ルトだった Minitest のファイルは test ディレクトリに作成されなくなっています。その代わ り、RSpec のファイルが spec ディレクトリに作成されます。しかし、好みに応じてジェネレ ータの設定を変更することができます。たとえば、scaffold ジェネレータを使ってコードを 追加するときに気になるのは、本書であまり詳しく説明しない不要なスペックがたくさん作 られてしまう点かもしれません。そこで最初からそうしたファイルを作成しないようにして みましょう。
```

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

    # 追加
    config.generators do |g|
      g.test_framework :rspec,
        fixtures: false,
        view_specs: false,
        helper_specs: false,
        routing_specs: false
      g.factory_bot false
    end
    # ここまで
  end
end
```

## 3. モデルスペック

+ はじめに、モデルスペックには次のようなテストを含めましょう。<br>

+ https://github.com/JunichiIto/everydayrails-rspec-jp-2022/compare/02-setup...03-models <br>

```
・ 有効な属性で初期化された場合は、モデルの状態が有効 (valid)になっていること
・ バリデーションを失敗させるデータであれば、モデルの状態が有効になっていないこと
・ クラスメソッドとインスタンスメソッドが期待通りに動作すること
```

+ User モデルの要件

```rb:sample.rb
describe User do
  # 姓、名、メール、パスワードがあれば有効な状態であること it "is valid with a first name, last name, email, and password" # 名がなければ無効な状態であること
  it "is invalid without a first name"
  # 姓がなければ無効な状態であること
  it "is invalid without a last name"
  # メールアドレスがなければ無効な状態であること
  it "is invalid without an email address"
  # 重複したメールアドレスなら無効な状態であること
  it "is invalid with a duplicate email address"
  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string"
end
```

## モデルスペックを作成する

+ `$ bin/rails g rspec:model user`を実行<br>

+ `$ bundle exec rspec`を実行 (下記のようになればOK)<br>

```
User
  add some examples to (or delete) /Users/groovy/Documents/everydayrails-rspec-jp-2022-02-setup/spec/models/user_spec.rb (PENDING: Not yet implemented)

Pending: (Failures listed here are expected and do not affect your suite's status)

  1) User add some examples to (or delete) /Users/groovy/Documents/everydayrails-rspec-jp-2022-02-setup/spec/models/user_spec.rb
     # Not yet implemented
     # ./spec/models/user_spec.rb:4


Finished in 0.00466 seconds (files took 7.78 seconds to load)
1 example, 0 failures, 1 pending
```

+ `spec/models/user_spec.rb`を編集<br>

```rb:user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do
  # 編集
  # 姓、名、メール、パスワードがあれば有効な状態であること
  it "is valid with a first name, last name, email, and password"
  # 名がなければ無効な状態であること
  it "is invalid without a first name"
  # 姓がなければ無効な状態であること
  it "is invalid without a last name"
  # メールアドレスがなければ無効な状態であること
  it "is invalid without an email address"
  # 重複したアドレスなら無効な状態であること
  it "is invalid with a duplicate email address"
  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string"
  # ここまで
end
```

+ `$ bundle exec rspec`を実行<br>

６つの保留中(pending)のスペックができた<br>

```
User
  is valid with a first name, last name, email, and password (PENDING: Not yet implemented)
  is invalid without a first name (PENDING: Not yet implemented)
  is invalid without a last name (PENDING: Not yet implemented)
  is invalid without an email address (PENDING: Not yet implemented)
  is invalid with a duplicate email address (PENDING: Not yet implemented)
  returns a user's full name as a string (PENDING: Not yet implemented)

Pending: (Failures listed here are expected and do not affect your suite's status)

  1) User is valid with a first name, last name, email, and password
     # Not yet implemented
     # ./spec/models/user_spec.rb:5

  2) User is invalid without a first name
     # Not yet implemented
     # ./spec/models/user_spec.rb:7

  3) User is invalid without a last name
     # Not yet implemented
     # ./spec/models/user_spec.rb:9

  4) User is invalid without an email address
     # Not yet implemented
     # ./spec/models/user_spec.rb:11

  5) User is invalid with a duplicate email address
     # Not yet implemented
     # ./spec/models/user_spec.rb:13

  6) User returns a user's full name as a string
     # Not yet implemented
     # ./spec/models/user_spec.rb:15


Finished in 0.00339 seconds (files took 2.37 seconds to load)
6 examples, 0 failures, 6 pending
```

## RSpecの構文

古い構文<br>

```rb:spec.rb
# 2と1と足すと3になること
it "adds 2 and 1 to make 3" do
  (2 + 1).shoud eq 3
end
```

現行の `expect`構文ではテストする値を`expect()`メソッドに渡し、それに続けてマッチャを呼び出します。<br>

```rb:spec.rb
# 2 と 1 を足すと3になること
it "adds 2 and 1 to make 3" do
  expect(2 + 1).to eq 3
end
```

+ `spec/models/user_spec.rb`を編集<br>

```rb:user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do
  # 姓、名、メール、パスワードがあれば有効な状態であること
  # 追加
  it "is valid with a first name, last name, email, and password" do
    user = User.new(
      first_name: "Aaron",
      last_name: "Summer",
      email: "tester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )
    expect(user).to be_valid
  end
  # ここまで
  # 名がなければ無効な状態であること
  it "is invalid without a first name"
  # 姓がなければ無効な状態であること
  it "is invalid without a last name"
  # メールアドレスがなければ無効な状態であること
  it "is invalid without an email address"
  # 重複したアドレスなら無効な状態であること
  it "is invalid with a duplicate email address"
  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string"
end
```

```
上記はbe_valid という RSpec のマッチャを使って、モデルが有効な状態を 理解できているかどうかを検証しています。まずオブジェクトを作成し(このケースでは新 しく作られているが保存はされていない User クラスのインスタンスを作成し、user という 名前の変数に格納しています)、それからオブジェクトを expect に渡して、マッチャと比較 しています。
```

+ `$ bundle exec rspec`を実行<br>

```
User
  is valid with a first name, last name, email, and password
  is invalid without a first name (PENDING: Not yet implemented)
  is invalid without a last name (PENDING: Not yet implemented)
  is invalid without an email address (PENDING: Not yet implemented)
  is invalid with a duplicate email address (PENDING: Not yet implemented)
  returns a user's full name as a string (PENDING: Not yet implemented)

Pending: (Failures listed here are expected and do not affect your suite's status)

  1) User is invalid without a first name
     # Not yet implemented
     # ./spec/models/user_spec.rb:15

  2) User is invalid without a last name
     # Not yet implemented
     # ./spec/models/user_spec.rb:17

  3) User is invalid without an email address
     # Not yet implemented
     # ./spec/models/user_spec.rb:19

  4) User is invalid with a duplicate email address
     # Not yet implemented
     # ./spec/models/user_spec.rb:21

  5) User returns a user's full name as a string
     # Not yet implemented
     # ./spec/models/user_spec.rb:23


Finished in 0.08792 seconds (files took 2.33 seconds to load)
6 examples, 0 failures, 5 pending
```

## バリデーションをテストする

+ `spec/models/user_spec.rb`を編集<br>

```rb:user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do
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
  # 追加
  it "is invalid without a first name" do
    user = User.new(first_name: nil)
    user.valid?
    expect(user.errors[:first_name]).to include("can't be blank")
  end
  # ここまで
  # 姓がなければ無効な状態であること
  it "is invalid without a last name"
  # メールアドレスがなければ無効な状態であること
  it "is invalid without an email address"
  # 重複したアドレスなら無効な状態であること
  it "is invalid with a duplicate email address"
  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string"
end
```

+ `$ bundle exec rspec`を実行<br>

```
User
  is valid with a first name, last name, email, and password
  is invalid without a first name
  is invalid without a last name (PENDING: Not yet implemented)
  is invalid without an email address (PENDING: Not yet implemented)
  is invalid with a duplicate email address (PENDING: Not yet implemented)
  returns a user's full name as a string (PENDING: Not yet implemented)

Pending: (Failures listed here are expected and do not affect your suite's status)

  1) User is invalid without a last name
     # Not yet implemented
     # ./spec/models/user_spec.rb:21

  2) User is invalid without an email address
     # Not yet implemented
     # ./spec/models/user_spec.rb:23

  3) User is invalid with a duplicate email address
     # Not yet implemented
     # ./spec/models/user_spec.rb:25

  4) User returns a user's full name as a string
     # Not yet implemented
     # ./spec/models/user_spec.rb:27


Finished in 0.10259 seconds (files took 2.31 seconds to load)
6 examples, 0 failures, 4 pending
```

## 失敗するテストも書いてみる

+ `spec/models/user_spec.rb`を編集<br>

```rb:user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do
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
    expect(user.errors[:first_name]).to_not include("can't be blank") # to_notに変更してみる 通らなければOK　確認したら戻しておく
  end
  # 姓がなければ無効な状態であること
  it "is invalid without a last name"
  # メールアドレスがなければ無効な状態であること
  it "is invalid without an email address"
  # 重複したアドレスなら無効な状態であること
  it "is invalid with a duplicate email address"
  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string"
end
```

+ `$ bundle exec rspec`を実行<br>

```
User
  is valid with a first name, last name, email, and password
  is invalid without a first name (FAILED - 1)
  is invalid without a last name (PENDING: Not yet implemented)
  is invalid without an email address (PENDING: Not yet implemented)
  is invalid with a duplicate email address (PENDING: Not yet implemented)
  returns a user's full name as a string (PENDING: Not yet implemented)

Pending: (Failures listed here are expected and do not affect your suite's status)

  1) User is invalid without a last name
     # Not yet implemented
     # ./spec/models/user_spec.rb:21

  2) User is invalid without an email address
     # Not yet implemented
     # ./spec/models/user_spec.rb:23

  3) User is invalid with a duplicate email address
     # Not yet implemented
     # ./spec/models/user_spec.rb:25

  4) User returns a user's full name as a string
     # Not yet implemented
     # ./spec/models/user_spec.rb:27


Failures:

  1) User is invalid without a first name
     Failure/Error: expect(user.errors[:first_name]).to_not include("can't be blank")
       expected ["can't be blank"] not to include "can't be blank"
     # ./spec/models/user_spec.rb:18:in `block (2 levels) in <top (required)>'

Finished in 0.10124 seconds (files took 2.44 seconds to load)
6 examples, 1 failure, 4 pending

Failed examples:

rspec ./spec/models/user_spec.rb:15 # User is invalid without a first name
```

```
もうひとつ、アプリケーション側のコードを変更して、テストの実行結果にどんな変化が 起きるか確認する方法もあります。先ほどのテストコードの変更を元に戻し(to_not を to に戻す)、それから User モデルを開いて first_name のバリデーションをコメントアウトし てください。
```

+ `app/models/user.rb`を編集<br>

```rb:user.rb
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable, :trackable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :trackable, :validatable

  # validates :first_name, presence: true # コメントアウトしてみる
  validates :last_name, presence: true

  has_many :projects, dependent: :destroy
  has_many :notes, dependent: :destroy

  before_save :ensure_authentication_token
  after_create :send_welcome_email

  def name
    [first_name, last_name].join(" ")
  end

  geocoded_by :last_sign_in_ip do |user, result|
    if !user.local? && geocode = result.first
      user.location = "#{geocode.city}, #{geocode.state}, #{geocode.country}"
      user.save
    end
  end

  def local?
    ["localhost", "127.0.0.1", "0.0.0.0"].include? last_sign_in_ip
  end

  def after_database_authentication
    GeocodeUserJob.perform_later self
  end

  private

  def ensure_authentication_token
    if authentication_token.blank?
      self.authentication_token = generate_authentication_token
    end
  end

  def generate_authentication_token
    loop do
      token = Devise.friendly_token
      break token unless User.where(authentication_token: token).first
    end
  end

  def send_welcome_email
    UserMailer.welcome_email(self).deliver_later
  end
end
```

+ `$ bundle exec rspec`を実行<br>

+ `確認後 `user.rb`のコメントアウトを戻しておく<br>

## last_nameのバリデーションをアプローチ

+ `spec/models/user_spec.rb`を編集<br>

```rb:user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do
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
  # 追加
  it "is invalid without a last name" do
    user = User.new(last_name: nil)
    user.valid?
    expect(user.errors[:last_name]).to include("can't be blank")
  end
  # ここまで
  # メールアドレスがなければ無効な状態であること
  it "is invalid without an email address"
  # 重複したアドレスなら無効な状態であること
  it "is invalid with a duplicate email address"
  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string"
end
```

+ `$ bundle exec rspec`を実行<br>

```
User
  is valid with a first name, last name, email, and password
  is invalid without a first name
  is invalid without a last name
  is invalid without an email address (PENDING: Not yet implemented)
  is invalid with a duplicate email address (PENDING: Not yet implemented)
  returns a user's full name as a string (PENDING: Not yet implemented)

Pending: (Failures listed here are expected and do not affect your suite's status)

  1) User is invalid without an email address
     # Not yet implemented
     # ./spec/models/user_spec.rb:27

  2) User is invalid with a duplicate email address
     # Not yet implemented
     # ./spec/models/user_spec.rb:29

  3) User returns a user's full name as a string
     # Not yet implemented
     # ./spec/models/user_spec.rb:31


Finished in 0.07742 seconds (files took 2.37 seconds to load)
6 examples, 0 failures, 3 pending
```

## email属性のユニークバリデーションをテスト

+ `spec/models/user_spec.rb`を編集<br>

```rb:user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do
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
  it "is invalid without an email address"
  # 重複したアドレスなら無効な状態であること
  # 編集
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
  # ここまで
  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string"
end
```

+ `$ bundle exec rspec`を実行<br>

```
User
  is valid with a first name, last name, email, and password
  is invalid without a first name
  is invalid without a last name
  is invalid without an email address (PENDING: Not yet implemented)
  is invalid with a duplicate email address
  returns a user's full name as a string (PENDING: Not yet implemented)

Pending: (Failures listed here are expected and do not affect your suite's status)

  1) User is invalid without an email address
     # Not yet implemented
     # ./spec/models/user_spec.rb:27

  2) User returns a user's full name as a string
     # Not yet implemented
     # ./spec/models/user_spec.rb:46


Finished in 0.10949 seconds (files took 2.27 seconds to load)
6 examples, 0 failures, 2 pending
```
