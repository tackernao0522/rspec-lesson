## 02 Setup

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
