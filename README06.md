# 5. コントローラスペック p78〜

```
Railsチームについて私がすごいと思うのは、フレームワークでもういらないと思われる機能はどんどん切り捨てていく精神です。
Rails5.0では過剰に使われていた二つのテストヘルパー(訳注:assignsメソッドと assert_template メソッド)がクビになりました。さらに、コントローラ層のテストが公式に格下げ(正式な用語を使うなら soft-deprecated )されました。
正直にいうと、私はここ数年、自分のアプリケーションであまりコントローラのテストを書いてきませんでした。
コントローラのテストはすぐ壊れやすくなりますし、アプリケーション内の他の実装の詳細へ過剰に焦点が当てられることも多いです。
RailsチームとRSpecチームの双方が、コントローラのテスト(機能テスト層とも呼ばれま す)を削除するか、またはモデルのテスト(単体テスト)か、より高いレベルの統合テストと置き換えることを推奨しています。
こんな話を聞くと気分が滅入ってしまう人もいるかもしれませんが、心配はいりません。この変化は状況を改善するための変化です!みなさんは モデルをテストする方法はすでに理解していますし、統合テストについても次の章以降で説明します。
ですが、コントローラのテストもまったく無視するわけにはいきません。なぜなら、移行 期間中はコントローラのテストもちゃんと意味をもって存在しているからです。
また、私はなぜ Railsチームがこのレベルのテストを格下げしたのか、その理由を理解することも重要 だと考えています。
加えて、もしあなたに RSpec でテストされているレガシーなRailsアプリケーションを保守する機会があれば、コントローラスペックを見かけることもきっとあると思います。
```

この章ではコントローラのテストの基礎について次のようなことを学びます。<br>

• まず、コントローラのテストとモデルのテストの違いを確認します。<br>
• それからコントローラのアクションをいくつかテストします。<br>
• 次に、認証が必要なアクションについて説明します。<br>
• そのあとで、ユーザーの入力をテストします。入力値が不正な場合もテストします。<br>
• 最後に CSV や JSON のような非 HTML の出力を持つコントローラのメソッドをテストします。<br>

+ [サンプルコード](https://github.com/JunichiIto/everydayrails-rspec-jp-2022/tree/05-controllers) <br>

## コントローラスペックの基本

```
では一番シンプルなコントローラから始めましょう。Homeコントローラは一つの仕事しかしません。
まだサインインしていない人のために、アプリケーションのホームページを返す仕事です。
```

+ `$ bin/rails g rspec:controller home --controller-specs --no-request-specs`<br>

```
以前は bin/rails g rspec:controller home というコマンドでコントローラのテストが作成できましたが、RSpec Rails 4.0.0以降ではリクエストスペックが優先的に作成されるようになったため、--controller-specs --no-request-specs というオプションを付ける必要があります。なお、リクエストスペックについては[第7章]で説明します。
```

+ `app/controllers/home_controller.rb`のコード<br>

```rb:home_controller.rb
class HomeController < ApplicationController

  skip_before_action :authenticate_user!

  def index
  end
end
```

+ `spec/controllers/home_controller_spec.rb`を編集<br>

```rb:home_controller_spec.rb
require 'rails_helper'

RSpec.describe HomeController, type: :controller do
  describe "#index" do
    # 正常にレスポンスを返すこと
    it "response successfully" do
      get :index
      expect(response).to be_successful
    end
  end
end
```

```
response はブラウザに返すべきアプリケーションの全データを保持しているオブジェクトです。
この中には HTTP レスポンスコードも含まれます。be_successful はレスポンスステータスが成功(200レスポンス)か、それ以外(たとえば500エラー)であるかをチェックします。
```

+ `$ bundle exec rspec spec/controllers`を実行<br>

```
HomeController
  #index
    response successfully

Finished in 0.0781 seconds (files took 2.21 seconds to load)
1 example, 0 failures
```

```
テストが正しく機能していることを確認するため、わざとテストを失敗させましょう。
最も簡単な方法は expect(response).to be_successful の to を to_not に変えて、レスポンス が成功しないように期待することです。
```

+ `spec/controllers/home_controller_spec.rb`を編集<br>

```rb:home_controller_spec.rb
require 'rails_helper'

RSpec.describe HomeController, type: :controller do
  describe "#index" do
    # 正常にレスポンスを返すこと
    it "response successfully" do
      get :index
      expect(response).to_not be_successful # 編集 確認後元に戻す
    end
  end
end
```

+ `$ bundle exec rspec spec/controllers`を実行<br>

```
Failures:

  1) HomeController#index response successfully
     Failure/Error: expect(response).to_not be_successful
       expected `#<ActionDispatch::TestResponse:0x0000000112810d18 @mon_data=#<Monitor:0x0000000112810cc8>, @mon_data_...e, @cache_control={}, @request=#<ActionController::TestRequest GET "http://test.host/" for 0.0.0.0>>.successful?` to be falsey, got true
     # ./spec/controllers/home_controller_spec.rb:8:in `block (3 levels) in <top (required)>'

Finished in 0.10148 seconds (files took 2.22 seconds to load)
1 example, 1 failure

Failed examples:

rspec ./spec/controllers/home_controller_spec.rb:6 # HomeController#index response successfully
```

```
特定の HTTP レスポンスコードが返ってきているかどうかも確認できます。
この場合であれば200 OK のレスポンスが返ってきてほしいはずです。
```

```rb:home_controller_spec.rb
require 'rails_helper'

RSpec.describe HomeController, type: :controller do
  describe "#index" do
    # 正常にレスポンスを返すこと
    it "response successfully" do
      get :index
      expect(response).to be_successful
    end

    # 追加
    it "returns a 200 response" do
      get :index
      expect(response).to have_http_status "200"
    end
    # ここまで
  end
end
```

+ `$ bundle exec rspec spec/controllers`を実行<br>

```
HomeController
  #index
    response successfully
    returns a 200 response

Finished in 0.08741 seconds (files took 2.18 seconds to load)
2 examples, 0 failures
```

```
これもちゃんとパスします。
繰り返しになりますが、こんなテストは一見すると退屈に見えます。
ですが、面白い要素 も実は隠れています。コントローラをもう一度見てください。
よく見ると、アプリケーショ ン全体で使われているユーザー認証用の before_action をスキップしていますね。
この行を コメントアウトしてテストを実行すると、何が起きるでしょうか?
```

+ `app/controllers/home_controller.rb`を編集<br>

```rb:home_controller.rb
class HomeController < ApplicationController

  # skip_before_action :authenticate_user! コメントアウトしてみる 確認したら戻しておく

  def index
  end
end
```

+ `$ bundle exec rspec spec/controllers`を実行<br>

```
HomeController
  #index
    response successfully (FAILED - 1)
    returns a 200 response (FAILED - 2)

Failures:

  1) HomeController#index response successfully
     Failure/Error: get :index

     Devise::MissingWarden:
       Devise could not find the `Warden::Proxy` instance on your request environment.
       Make sure that your application is loading Devise and Warden as expected and that the `Warden::Manager` middleware is present in your middleware stack.
       If you are seeing this on one of your tests, ensure that your tests are either executing the Rails middleware stack or that your tests are using the `Devise::Test::ControllerHelpers` module to inject the `request.env['warden']` object for you.
     # ./spec/controllers/home_controller_spec.rb:7:in `block (3 levels) in <top (required)>'

  2) HomeController#index returns a 200 response
     Failure/Error: get :index

     Devise::MissingWarden:
       Devise could not find the `Warden::Proxy` instance on your request environment.
       Make sure that your application is loading Devise and Warden as expected and that the `Warden::Manager` middleware is present in your middleware stack.
       If you are seeing this on one of your tests, ensure that your tests are either executing the Rails middleware stack or that your tests are using the `Devise::Test::ControllerHelpers` module to inject the `request.env['warden']` object for you.
     # ./spec/controllers/home_controller_spec.rb:12:in `block (3 levels) in <top (required)>'

Finished in 0.07694 seconds (files took 2.2 seconds to load)
2 examples, 2 failures

Failed examples:

rspec ./spec/controllers/home_controller_spec.rb:6 # HomeController#index response successfully
rspec ./spec/controllers/home_controller_spec.rb:11 # HomeController#index returns a 200 response
```

```
興味深いことに、Devise のテスト用ヘルパーが見つからないためにテストが失敗しました (訳注: 失敗メッセージの中に Devise::Test::ControllerHelpers を使っていることを確認してください、という内容が書いてあります)。
この結果からわかることは、コントローラのskip_before_action の行はちゃんと仕事をしていたということです!
このあと、コントローラをテストする際にコントローラが認証済み なるようにテストを修正します。
ですが、いったんはコメントアウトした行を元に戻し、Home コントローラの機能を元に戻してください。
```

## 認証が必要なコントローラスペック p84〜

+ `$ bin/rails g rspec:controller projects --controller-specs --no-request-specs`を実行<br>

+ `$ spec/controllers/projects_controller_spec.rb`を編集<br>

```rb:projects_controller_spec.rb
require 'rails_helper'

RSpec.describe ProjectsController, type: :controller do
  describe "#index" do
    # 正常にレスポンスを返すこと
    it "response successfully" do
      get :index
      expect(response).to be_successful
    end

    # 200レスポンスを返すこと
    it "returns a 200 response" do
      get :index
      expect(response).to have_http_status "200"
    end
  end
end
```

```
スペックを実行すると、先ほども出てきた Devise のヘルパーが見つからないというメッセージが表示されます。
それでは今からこの問題に対処していきましょう。Devise は認証が必 要なコントローラのアクションに対して、ユーザーのログイン状態をシミュレートするヘルパーを提供しています。
ですが、そのヘルパーはまだ追加されていません。失敗メッセージ にはこの問題に対処する方法が少し詳しく載っています。
というわけで、テストスイートにこのヘルパーモジュールを組み込みましょう。
spec/rails_helper.rb を開き、次のような設定を 追加してください。
```

+ `spec/rails_helper.rb`を編集<br>

```rb:rails_helper.rb
# This file is copied to spec/ when you run 'rails generate rspec:install'
require 'spec_helper'
ENV['RAILS_ENV'] ||= 'test'
require File.expand_path('../config/environment', __dir__)
# Prevent database truncation if the environment is production
abort("The Rails environment is running in production mode!") if Rails.env.production?
require 'rspec/rails'
# Add additional requires below this line. Rails is not loaded until this point!

# Requires supporting ruby files with custom matchers and macros, etc, in
# spec/support/ and its subdirectories. Files matching `spec/**/*_spec.rb` are
# run as spec files by default. This means that files in spec/support that end
# in _spec.rb will both be required and run as specs, causing the specs to be
# run twice. It is recommended that you do not name files matching this glob to
# end with _spec.rb. You can configure this pattern with the --pattern
# option on the command line or in ~/.rspec, .rspec or `.rspec-local`.
#
# The following line is provided for convenience purposes. It has the downside
# of increasing the boot-up time by auto-requiring all files in the support
# directory. Alternatively, in the individual `*_spec.rb` files, manually
# require only the support files necessary.
#
# Dir[Rails.root.join('spec', 'support', '**', '*.rb')].sort.each { |f| require f }

# Checks for pending migrations and applies them before tests are run.
# If you are not using ActiveRecord, you can remove these lines.
begin
  ActiveRecord::Migration.maintain_test_schema!
rescue ActiveRecord::PendingMigrationError => e
  puts e.to_s.strip
  exit 1
end
RSpec.configure do |config|
  # Remove this line if you're not using ActiveRecord or ActiveRecord fixtures
  config.fixture_path = "#{::Rails.root}/spec/fixtures"

  # If you're not using ActiveRecord, or you'd prefer not to run each of your
  # examples within a transaction, remove the following line or assign false
  # instead of true.
  config.use_transactional_fixtures = true

  # You can uncomment this line to turn off ActiveRecord support entirely.
  # config.use_active_record = false

  # RSpec Rails can automatically mix in different behaviours to your tests
  # based on their file location, for example enabling you to call `get` and
  # `post` in specs under `spec/controllers`.
  #
  # You can disable this behaviour by removing the line below, and instead
  # explicitly tag your specs with their type, e.g.:
  #
  #     RSpec.describe UsersController, type: :controller do
  #       # ...
  #     end
  #
  # The different available types are documented in the features, such as in
  # https://relishapp.com/rspec/rspec-rails/docs
  config.infer_spec_type_from_file_location!

  # Filter lines from Rails gems in backtraces.
  config.filter_rails_from_backtrace!
  # arbitrary gems may also be filtered via:
  # config.filter_gems_from_backtrace("gem name")

  # 追加
  # コントローラスペックで Devise のテストヘルパーを使用する
  config.include Devise::Test::ControllerHelpers, type: :controller
  # ここまで
end
```

```
この状態でテストを実行してもまだ失敗します。
ですが、失敗メッセージには新しい情報が載っています。
つまり、ちょっとは前に進んだということです。どちらのテストも基本的 に同じ理由、すなわち、成功を表す 200 レスポンスではなく、リダイレクトを表す 302 レス ポンスが返ってきているために失敗しているのです。
失敗するのは index アクションがユーザーのログインを要求しているにもかかわらず、私たちはまだそれをテスト内でシミュレートしていないからです。
```

※ <br>

```
みなさんがもし Devise を使っていないのであれば、使用している認証ライブラリ のドキュメントを読み、コントローラスペックでログイン状態をシミュレートす るにはどういう方法が良いのかを確認してください。もし Rails が提供している has_secure_password メソッドなどを使って認証機能を自分で作っている場合は、次 のようにして自分でヘルパーメソッドを定義してみてください。

# 自分で対処する ....
def sign_in(user)
  cookies[:auth_token] = user.auth_token
end
```

```
すでに Devise のヘルパーはテストスイートに組み込んであるので、ログイン状態をシミュ レートすることができます。
具体的にはテストユーザーを作成し、それからそのユーザーでログインするようにテストに伝えます。
テストユーザーは両方のテストで有効になるよう before ブロックで作成し、それからログイン状態をシミュレートするために sign_in ヘルパーを使います。
```

+ `$ spec/controllers/projects_controller_spec.rb`を編集<br>

```rb:projects_controller_spec.rb
require 'rails_helper'

RSpec.describe ProjectsController, type: :controller do
  describe "#index" do
    # 追加
    before do
      @user = FactoryBot.create(:user)
    end
    # ここまで

    # 正常にレスポンスを返すこと
    it "response successfully" do
      sign_in @user # 追加
      get :index
      expect(response).to be_successful
    end

    # 200レスポンスを返すこと
    it "returns a 200 response" do
      sign_in @user # 追加
      get :index
      expect(response).to have_http_status "200"
    end
  end
end
```

+ `$ bundle exec rspec spec/controllers`を実行<br>

```
さあ、これでスペックはパスするはずです。なぜなら、index アクションには認証済みのユーザーでアクセスしていることになるからです。
ここでちょっとテストをパスさせるためにどうしたのかを考えてみましょう。
テストがパスしたのは必要な変更を加えたあとです。最初はログインしていなかったので、テストは失敗していました。
アプリケーションセキュリティの観点からすると、認証されていないユーザー(ゲストと呼んでもいいでしょう)がアクセスしたら強制的にリダイレクトされることもテストすべきではないでしょうか。
ここからテストを拡張して、このシナリオを追加することは可能です。
こういうケースは describe と context のブロックを使うと、テストを整理しやすくなります。
というわけで、次のように変更してみましょう。
```

+ `$ spec/controllers/projects_controller_spec.rb`を編集<br>

```rb:projects_controller_spec.rb
require 'rails_helper'

RSpec.describe ProjectsController, type: :controller do
  describe "#index" do
    # 認証済みユーザーとして
    context "as an authenticated user" do # 追加
      before do
        @user = FactoryBot.create(:user)
      end

      # 正常にレスポンスを返すこと
      it "response successfully" do
        sign_in @user
        get :index
        expect(response).to be_successful
      end

      # 200レスポンスを返すこと
      it "returns a 200 response" do
        sign_in @user
        get :index
        expect(response).to have_http_status "200"
      end
    end # 追加

    # 追加
    # ゲストとして
    context "as a guest" do
      # テストをここに書く
    end
    # ここまで
  end
end
```

```
ここでは index アクションの describe ブロック内に、二つの context を追加しました。
一つ目は認証済みのユーザーを扱う context です。
テストユーザーを作成する before ブロック がこの context ブロックの内側で入れ子になっている点に注意してください。
それから、スペックを実行して正しく変更できていることを確認してください。
```

+ `$ bundle exec rspec spec/controllers`を実行<br>

```
HomeController
  #index
    response successfully
    returns a 200 response

ProjectsController
  #index
    as an authenticated user
      response successfully
      returns a 200 response

Finished in 0.23497 seconds (files took 2.22 seconds to load)
4 examples, 0 failures
```

```
続いて認証されていないユーザーの場合をテストしましょう。
"as a guest" の context を変更し、次のようなテストを追加してください。
```

+ `spec/controllers/projects_controller_spec.rb`を編集 p88<br>

```rb:projects_controller_spec.rb
require 'rails_helper'

RSpec.describe ProjectsController, type: :controller do
  describe "#index" do
    # 認証済みユーザーとして
    context "as an authenticated user" do
      before do
        @user = FactoryBot.create(:user)
      end

      # 正常にレスポンスを返すこと
      it "response successfully" do
        sign_in @user
        get :index
        expect(response).to be_successful
      end

      # 200レスポンスを返すこと
      it "returns a 200 response" do
        sign_in @user
        get :index
        expect(response).to have_http_status "200"
      end
    end

    # 編集
    # ゲストとして
    context "as a guest" do
      # 302レスポンスを返すこと
      it "returns a 302 response" do
        get :index
        expect(response).to have_http_status "302"
      end

      # サインイン画面にリダイレクトすること
      it "redirects to the sign-in page" do
        get :index
        expect(response).to redirect_to "/users/sign_in"
      end
    end
    # ここまで
  end
end
```

+ `bundle exec rspec spec/controllers`を実行 (すべて通る)<br>

```
最初のスペックは難しくないはずです。have_http_status マッチャはすでに使っていますし、
302 というレスポンスコードもちょっと前の失敗メッセージに出てきました。
二つ目のスペックでは redirect_to という新しいマッチャを使っています。ここではコントローラが 認証されていないリクエストの処理を中断し、Devise が提供しているログイン画面に移動さ せていることを検証しています。
同じテクニックはアプリケーションの認可機能(つまり、ログイン済みのユーザーが、や りたいことをできるかどうかの判断)にも適用できます。これはコントローラの3行目で処理されています。

before_action :project_owner?, except: %i[ index new create ]

このアプリケーションではユーザーがプロジェクトのオーナーであることを要求します。
では、新しいテストを追加しましょう。
今回は show アクションのテストです。一つの describe ブロックと二つの context ブロックを追加してください。
```

+ `spec/controllers/projects_controller_spec.rb`を編集 p88<br>

```rb:projects_controller_spec.rb
require 'rails_helper'

RSpec.describe ProjectsController, type: :controller do
  describe "#index" do
    # 認証済みユーザーとして
    context "as an authenticated user" do
      before do
        @user = FactoryBot.create(:user)
      end

      # 正常にレスポンスを返すこと
      it "response successfully" do
        sign_in @user
        get :index
        expect(response).to be_successful
      end

      # 200レスポンスを返すこと
      it "returns a 200 response" do
        sign_in @user
        get :index
        expect(response).to have_http_status "200"
      end
    end

    # ゲストとして
    context "as a guest" do
      # 302レスポンスを返すこと
      it "returns a 302 response" do
        get :index
        expect(response).to have_http_status "302"
      end

      # サインイン画面にリダイレクトすること
      it "redirects to the sign-in page" do
        get :index
        expect(response).to redirect_to "/users/sign_in"
      end
    end
  end

  # 追加
  describe "#show" do
    # 認可されたユーザーとして
    context "as an authorized user" do
      before do
        @user = FactoryBot.create(:user)
        @project = FactoryBot.create(:project, owner: @user)
      end

      # 正常にレスポンスを返すこと
      it "responds successfully" do
        sign_in @user
        get :show, params: { id: @project.id }
        expect(response).to be_successful
      end
    end

    # 認可されていないユーザーとして
    context "as an unauthorized user" do
      before do
        @user = FactoryBot.create(:user)
        other_user = FactoryBot.create(:user)
        @project = FactoryBot.create(:project, owner: other_user)
      end

      # ダッシュボードにリダイレクトすること
      it "redirects to the dashboard" do
        sign_in @user
        get :show, params: { id: @project.id }
        expect(response).to redirect_to root_path
      end
    end
  end
  # ここまで
end
```

+ `bundle exec rspec spec/controllers`を実行(パスする)<br>

```
今回はテストごとに @project を作成しました。
最初のcontextではログインしたユーザーがプロジェクトのオーナーになっています。
二つ目の context では別のユーザーがオーナーになっています。
このテストにはもう一つ新しい部分があります。
それはプロジェクトの id をコントローラアクションの param値として渡さなければいけない点です。
テストを実行してパスすることを確認してください。
```
