## ヘッドレスドライバを使う p118〜

テストの実行中にブラウザのウィンドウが開くのはあまり望ましくないケースがよくあります。<br>
たとえば、GitHub Actions や Travis CI、Jenkins のような継続的インテグレーション(CI) 環境で実行する場合、先ほど作ったテストは CLI(コマンドラインインターフェース)上で実 行する必要があります。<br>
ですが、CLI 上では新しいウィンドウを開くことはできません。<br>
こういった要件に対応するため、Capybara はヘッドレス ドライバを使えるようになっています。<br>
そこで Chrome のヘッドレスモードを使ってテストを実行するよう、spec/support/capybara.rb を編集して次のようにドライバを変更してください。

+ `spec/support/capybara.rb`を編集<br>

```rb:capybara.rb
RSpec.configure do |config|
  config.before(:each, type: :system) do
    driven_by :rack_test
  end

  config.before(:each, type: :system, js: true) do
    driven_by :selenium_chrome_headless # 編集
  end
end
```

さあこれでブラウザのウィンドウを開くことなく、JavaScript を使うテストを実行できるようになりました。<br>
実際にスペックを実行してみてください。<br>

+ `$ bundle exec rspec spec/system/tasks_spec.rb`を実行<br>

設定がうまくいっていれば、Chrome のウィンドウが表示されることなくテストが完了するはずです。<br>

## JavaScript の完了を待つ

デフォルトでは Capybara はボタンが現れるまで2秒待ちます。2秒待っても表示されなければ諦めます。<br>
次のようにするとこの秒数を好きな⻑さに変更できます。<br>

```rb:sample.rb
Capybara.default_max_wait_time = 15
```

上の設定では待ち時間を15秒に設定しています。<br>
この設定は spec/support/capybara.rb ファイルに書いてテストスイート全体に適用することができます(ただし、みなさんのアプリケーションが本書のサンプルアプリケーションと同じやり方で Capybara を設定していることが前提になります。<br>
つまり、このファイルは spec/rails_helper.rb によって読み込まれる場所に配置される必要があります)。<br>
しかし、この変更はテストスイートの実行がさらに遅くなる原因になるかもしれないので注意してください。<br>
もしこの設定を変えたいと思ったら、必要に応じてその都度 using_wait_time を使うようにした方がまだ良いかもしれません。<br>
たとえば次のようなコードになります。<br>

```rb:sample.rb
# 本当に遅い処理を実行する
scenario "runs a really slow process" do
  using_wait_time(15) do
    # テストを実行する
  end
end
```

いずれにしても基本的なルールとして、処理の完了を待つために Ruby の sleepメソッドを使うのは避けてください。<br>

## スクリーンショットを使ってデバッグする

システムスペックでは take_screenshot メソッドを使って、テスト内のあらゆる場所でシミュレート中のブラウザの画像を作成することができます。<br>
ただし、このメソッドは JavaScriptドライバ(本書でいうところの selenium-webdriver です)を使うテストでしか使用できない点に注意してください。<br>
画像ファイルはデフォルトで tmp/capybara に保存されます。<br>
また、テストが失敗したら、自動的にスクリーンショットが保存されます!<br>
この機能は ヘッドレスブラウザで実行している統合テストをデバッグするのに大変便利です。<br>

＊ JavaScriptドライバではなく Rack::Test を使ってテストしている場合は、<br>
本章の「シ ステムスペックをデバッグする」で説明したように save_page メソッドを使って HTML ファイルを tmp/capybara に保存したり、save_and_open_page メソッドを使ってファイルを自動的にブラウザで開いたりすることができます。<br>

## システムスペックとフィーチャースペック

システムスペックが導入されたのは RSpec Rails 3.7からです。<br>
そして、システムスペックは その背後で Rails 5.1から導入されたシステムテストを利用しています。<br>
それ以前はフィーチャスペック(feature specs)と呼ばれる RSpec Rails 独自の機能を使って統合テストを書いていました。<br>
フィーチャスペックは見た目も機能面もシステムスペックに非常によく似ています。 <br>
では、どちらのスペックを使うのが良いのでしょうか?もし、みなさんがまだ統合テストを一度も書いたことがないのなら、フィーチャスペックではなく、システムスペックを使ってください。<br>
ですが、フィーチャスペックも廃止されたわけではありません。昔から保守されている Rails アプリケーションではフィーチャスペックを使い続けている可能性もあります。<br>
そこで、このセクションでは簡単にシステムスペックとフィーチャスペックの違いを説明しておきます。<br>
たとえば、この章の最初に紹介したシステムスペックのコード例をフィーチャスペックを使って書いた場合は次のようなコードになります。<br>

+ `spec/features/projects_spec.rb`(例)<br>

```rb:projects_spec.rb
require 'rails_helper'

RSpec.feature "Projects", type: :feature do
  # ユーザーは新しいプロジェクトを作成する
  scenario "user creates a new project" do
    user = FactoryBot.create(:user)

    visit root_path
    click_link "Sign in"
    fill_in "Email", with: user.email
    fill_in "Password", with: user.password
    click_button "Log in"

    expect {
      click_link "New Project"
      fill_in "Name", with: "Test Project"
      fill_in "Description", with: "Trying out Capybara"
      click_button "Create Project"

      expect(page).to have_content "Project was successfully created"
      expect(page).to have_content "Test Project"
      expect(page).to have_content "Owner: #{user.name}"
    }.to change(user.projects, :count).by(1)
  end
end
```

一見、システムスペックとほとんど違いがありませんが、次のような点が異なります。<br>

+ spec/system ではなく spec/features にファイルを保存する<br>
+ describe メソッドではなく feature メソッドを使う<br>
+ type: オプションに :system ではなく :feature を指定する<br>

ですが、scenario メソッドの内部はシステムスペックと全く同じです。<br>
このほかにもフィーチャスペックには以下のような違いがあります。<br>

+ let や let! のエイリアスとして given や given! が使える(let や let! については第8章で説明します)<br>
+ before のエイリアスとして background が使える<br>
+ スクリーンショットを撮る場合、save_screenshotは使えるが、take_screenshotは使えない<br>
+ テストが失敗してもスクリーンショットは自動的に保存されない(明示的にsave_-screenshot メソッドを呼び出す必要があります)<br>

フィーチャスペックはまだ使えますが、レガシーな機能になりつつあるため、早めにシステムスペックに移行する方が良いと思いす。<br>
フィーチャスペックをシステムスペックに移行する場合は次のような手順に従ってください。<br>

1. Rails 5.1以上かつ、RSpec Rails 3.7以上になっていることを確認する<br>
2. システムスペックで使用する Capybara、Selenium Webdriver、Webdrivers といった gem はなるべく最新のものを使うようにアップデートする<br>
3. js: true のタグが指定された場合にドライバが切り替わるように設定を変更する(この章で紹介した spec/support/capybara.rb を参照してください)<br>
4. spec/features ディレクトリを spec/system にリネームする<br>
5. 各スペックのタイプをtype::featureからtype::systemに変更する<br>
6. 各スペックで使われているfeatureをdescribeに変更する<br>
7. 各スペックで使われているbackgroundをbeforeに変更する<br>
8. 各スペックで使われているgiven/given!をlet/let!に変更する<br>
9. 各スペックで使われているscenarioをitに変更する(この変更は任意です)<br>
10. spec/rails_helper.rbのconfig.includeなどで、type::featureになっている設定があれば type: :system に変更する<br>

移行作業が終わったらテストスイートを実行してみてください。移行後のシステムスペックはきっとパスするはずです<br>

## まとめ

システムスペックの書き方は一つ前の章までに習得したスキルの上に成り立っています。 <br>
また、システムスペックは習得するのも理解するのも比較的簡単です。<br>
なぜならWebブラウザを起動すれば操作をシミュレートするために必要なステップが簡単にわかりますし、その次に Capybaraを使ってそのステップを再現すれば済むからです。<br>
これは別にズルをしているわけではありません! このアプローチはみなさんがテストの書き方を練習し、コードのカバレッジを増やすための完璧な方法です。<br>
何もしなければコードはテストされないまま放置されてしまいます。<br>
さて、次の章では人間以外のユーザーとアプリケーションのやりとりをテストし、外部向 け API のカバレッジを増やしていきます。<br>

## 演習問題

+ システムスペックをいくつか書いてパスさせてください! 最初はシンプルなユーザーの操作から始め、テストを書くプロセスに慣れてきたら、より複雑な操作へと進みましょう。<br>

+ システムの example に必要なステップを書くときは、ユーザーのことを考えてください。<br>
ユーザーは何らかの必要があってそのステップをブラウザ上で操作する人々です。<br>
もっとシンプルにできそうな、もしくは削除しても大丈夫そうなステップはありませんか?<br>
そうすることでユーザー体験全般をもっと快適にすることはできませんか?<br>

# 7章 リクエストスペックでAPIをテストする p122〜

最近では Railsアプリケーションが外部向けAPIを持つことも増えてきました。<br>
たとえば、RailsアプリケーションはJavaScriptで作られたフロントエンドやネイティブモバイルアプリケーション、サードパーティ製アドオンのバックエンドとしてAPIを提供することがあります。<br>

こうしたAPIはこれまでにテストしてきたサーバーサイド出力による UI に追加される形で提供されることもありますし、UIのかわりに提供されることもあります。<br>
そして、みなさんのような開発者が APIを利用し、顧客は APIが高い信頼性を持っていることを望みます。<br>
ですのでやはり、みなさんは APIもテストしたくなるはずです!<br>

堅牢で開発者に優しい APIの作り方は本書の範疇を超えてしまいますが、APIのテストについてはそうではありません。<br>
嬉しいことに、もしみなさんがここまでにコントローラスペックやシステムスペックの章を読んできたのであれば、APIをテストするために必要な基礎知識はすでに習得しています。<br>
この章で説明する内容は以下のとおりです。<br>

+ リクエストスペックとシステムスペックの違い<br>
+ 様々な種類の RESTful な API リクエストをテストする方法<br>
+ コントローラスペックをリクエストスペックで置き換える方法<br>

## リクエストスペックとシステムスペックの比較

最初に、こうしたテストはどのように使い分けるべきなのでしょうか?<br>
第5章で説明したとおり、JSON(または XML)の出力はコントローラスペックで直接テストすることができます。<br>
みなさん自身のアプリケーションでしか使われない専用のシンプルなメソッドであれば、この方法で十分かもしれません。<br>
一方、より堅牢なAPIを構築するのであれば、第6章で説明したシステムスペックによく似た統合テストが必要になってきます。<br>
ですが、違うところもいくつかあります。RSpecの場合、今回の新しいAPI関連のテストは spec/requests ディレクトリに配置するのがベストです。<br>

これまでに書いたシステムスペックとは区別しましょう。リクエストスペックでは Capybara も使いません。<br>
Capybaraはブラウザの操作をシミュレートするだけであり、プログラム上のやりとりは特にシミュレートしないからです。<br>
かわりに、コントローラのレスポンスをテストする際に使ったHTTP動詞に対応するメソッド(get 、post 、delete 、patch)を使います。<br>

本書のサンプルアプリケーションにはユーザーのプロジェクト一覧にアクセスしたり、新しいプロジェクトを作成したりするための簡単なAPIが含まれています。<br>
どちらのエンド ポイントもトークンによる認証を使います。サンプルコードは app/controllers/api/projects_- controller.rb で確認できます。<br>
あまり難しいことはやっていませんが、先ほども述べたとおり、本書はテストの本であって、堅牢な APIを設計するための本ではありません。<br>

## GET リクエストをテストする p124〜

最初の例では、最初に紹介したエンドポイントにフォーカスします。<br>
このエンドポイントは認証完了後、クライアントにユーザーのプロジェクト一覧を含む JSONデータを返します。<br>
RSpecにはリクエストスペック用のジェネレータがあるので、これを使ってどういったコードが作成されるのか見てましょう。<br>
コマンドラインから次のコマンドを実行してください。<br>

+ `$ bin/rails g rspec:request projects_api`を実行<br>

+ `$ mv spec/requests/projects_apis_spec.rb spec/requests/projects_api_spec.rb`を実行してリネーム<br>

+ `spec/requests/projects_api_spec.rb`を編集<br>

```rb:projects_api_spec.rb
require 'rails_helper'

RSpec.describe "ProjectsApis", type: :request do
  # 1件のプロジェクトを読み出すこと
  it 'loads a project' do
    user = FactoryBot.create(:user)
    FactoryBot.create(:project,
      name: "Sample Project")
    FactoryBot.create(:project,
      name: "Second Sample Project",
      owner: user)

    get api_projects_path, params: {
      user_email: user.email,
      user_token: user.authentication_token
    }

    expect(response).to have_http_status(:success)
    json = JSON.parse(response.body)
    expect(json.length).to eq 1
    project_id = json[0]["id"]

    get api_project_path(project_id), params: {
      user_email: user.email,
      user_token: user.authentication_token
    }

    expect(response).to have_http_status(:success)
    json = JSON.parse(response.body)
    expect(json["name"]).to eq "Second Sample Project"
    # などなど
  end
end
```

+ `$ bundle exec rspec spec/requests`を実行するとパスする<br>

上のサンプルコードはコントローラスペックっぽさが薄れ、システムスペックっぽいパターンになっています。<br>
この新しいスペックがリクエストスペックです。最初はサンプルデータを作成しています。<br>
ここでは1人のユーザーと2件のプロジェクトを作成しています。一方のプロジェクトは先ほどのユーザーがオーナーで、もう一つのプロジェクトは別のユーザーがオーナーになっています。<br>

次に、HTTP GET を使ったリクエストを実行しています。コントローラスペックと同様、ルーティング名に続いてパラメー(params) を渡しています。<br>
この API ではユーザーのメールアドレスとサインインするためのトークンが必要になります。<br>
パラメータにはこの二つの値を含めています。ですが、コントローラスペックとは異なり、今回は好きなルーティング名を何でも使うことができます。<br>
リクエストスペックはコントローラに結びつくことはありません。これはコントローラスペックとは異なる点です。<br>
なので、テストしたいルーティング名をちゃんと指定しているか確認する必要も出てきます。<br>

それから、テストは返ってきたデータを分解し、取得結果を検証します。データベースには2件のプロジェクトが格納されていますが、<br>
このユーザーがオーナーになっているのは1件だけです。そのプロジェクトの ID を取得し、2番目の API コールでそれを利用しす。<br>
この API は1件のプロジェクトに対して、より多くの情報を返すエンドポイントです。この API は コールするたびに再認証が必要になる点に注意してください。<br>
ですので、メールアドレスと トークンは毎回パラメータとして渡す必要があります。<br>
最後に、この API コールで返ってきた JSON データをチェックし、そのプロジェクト名とテストデータのプロジェクト名が一致するか検証しています。そしてここではちゃんと一致します。<br>

## POSTリクエストをテストする

次のサンプルコードではAPIにデータを送信しています。<br>

+ `spec/requests_api_spec.rb`を編集<br>

```rb:requests_api_spec.rb
require 'rails_helper'

RSpec.describe "ProjectsApis", type: :request do
  # 1件のプロジェクトを読み出すこと
  it 'loads a project' do
    user = FactoryBot.create(:user)
    FactoryBot.create(:project,
      name: "Sample Project")
    FactoryBot.create(:project,
      name: "Second Sample Project",
      owner: user)

    get api_projects_path, params: {
      user_email: user.email,
      user_token: user.authentication_token
    }

    expect(response).to have_http_status(:success)
    json = JSON.parse(response.body)
    expect(json.length).to eq 1
    project_id = json[0]["id"]

    get api_project_path(project_id), params: {
      user_email: user.email,
      user_token: user.authentication_token
    }

    expect(response).to have_http_status(:success)
    json = JSON.parse(response.body)
    expect(json["name"]).to eq "Second Sample Project"
    # などなど
  end

  # 追加
  # プロジェクトを作成できること
  it 'creates a project' do
    user = FactoryBot.create(:user)

    project_attributes = FactoryBot.attributes_for(:project)

    expect {
      post api_projects_path, params: {
        user_email: user.email,
        user_token: user.authentication_token,
        project: project_attributes
      }
    }.to change(user.projects, :count).by(1)

    expect(response).to have_http_status(:success)
  end
  # ここまで
end
```

パスする<br>

やはりここでもサンプルデータの作成から始まっています。今回は1人のユーザーと有効なプロジェクトの属性を集めたハッシュが必要です。<br>
それからアクションを実行して期待どおりの変化が発生するかどうか確認しています。<br>
この場合はユーザーが持つ全プロジェクトの件数が1件増えることを確認します。<br>

今回のアクションはプロジェクト API に POST リクエストを送信することです。認証用のパラメータを送信する点は GET リクエストの場合と同じですが、今回はさらにプロジェクトの属性も含んでいます。<br>
それから最後にレスポンスのステータスをチェックしています。<br>

## コントローラスペックをリクストスペックで置き換える


ここまで見てきたサンプルコードでは API をテストすることにフォーカスしていました。<br>
しかし API に限らず、第5章で作成したコントローラスペックをリクエストスペックで置き換えることも可能です。<br>
既存の Home コントローラのスペックを思い出してください。このスペックは簡単にリクエストスペックに置き換えることができます。<br>
spec/requests/home_spec.rb にリクエストスペックを作成し、次のようなコードを書いてください。p128<br>

+ `$ bin/rails g rspec:request home`を実行<br>

+ `$ mv spec/requests/homes_spec.rb spec/requests/home_spec.rb`を実行してリネーム<br>

+ `spec/requests/home_spec.rb`を編集<br>

```rb:home_spec.rb
require 'rails_helper'

RSpec.describe "Home page", type: :request do
  # 正常なレスポンスを返すこと
  it "response successfully" do
    get root_path
    expect(response).to be_successful
    expect(response).to have_http_status "200"
  end
end
```

+ パスする<br>

もう少し複雑な例も見てみましょう。たとえば Project コントローラの create アクションのテストは次のようなリクエストスペックに書き換えることができます。<br>
spec/requests/projects_- spec.rb を作成してテストコードを書いてみましょう(前述の projects_api_spec.rb とはファイル名が異なる点に注意してください)。<br>

+ `$ bin/rails g rspec:request project`を実行<br>

+ `spec/requests/projects_spec.rb`を編集<br>

```rb:projects_spec.rb
require 'rails_helper'

RSpec.describe "Projects", type: :request do
  # 認証済みのユーザーとして
  context "as an authenticated user" do
    before do
      @user = FactoryBot.create(:user)
    end

    # 有効な属性値の場合
    context "with valid attributes" do
      # プロジェクトを追加できること
      it "adds a project" do
        project_params = FactoryBot.attributes_for(:project)
        sign_in @user
        expect {
          post projects_path, params: { project: project_params }
        }.to change(@user.projects, :count).by(1)
      end
    end

    # 無効な属性値の場合
    context "with invalid attributes" do
      # プロジェクトを追加できないこと
      it "does not add a project" do
        project_params = FactoryBot.attributes_for(:project, :invalid)
        sign_in @user
        expect {
          post projects_path, params: { project: project_params }
        }.to_not change(@user.projects, :count)
      end
    end
  end
end
```

コントローラスペックとの違いはごくわずかです。リクエストスペックでは Projectコントローラの create アクションに直接依存するのではなく、具体的なルーティング名を指定して POST リクエストを送信します。<br>
それ以外はコントローラスペックとまったく同じコードです。<br>

API用のコントローラとは異なり、このコントローラでは標準的なメールアドレスとパスワードの認証システムを使っています。<br>
なので、この仕組みがちゃんと機能するように、ちょっとした追加の設定がここでも必要になります。<br>
今回は Devise の sign_in ヘルパーをリクエストスペックに追加します。[Devise の wiki ページにあるサンプルコードを参考にして](https://github.com/heartcombo/devise/wiki/How-To:-sign-in-and-out-a-user-in-Request-type-specs-(specs-tagged-with-type:-:request))この設定を有効にしてみましょう。<br>
まず、spec/support/request_spec_helper.rb という新しいファイルを作成します。<br>

+ `$ touch spec/support/request_spec_helper.rb`を実行<br>

+ `spec/support/request_spec_helper.rb`を編集<br>

```rb:request_spec_helper.rb
module RequestSpecHelper
  include Warden::Test::Helpers

  def self.included(base)
    base.before(:each) { Warden.test_mode! }
    base.after(:each) { Warden.test_reset! }
  end

  def sign_in(resource)
    login_as(resource, scope: warden_scope(resource))
  end

  def sign_in(resource)
    login_as(resource, scope: warden_scope(resource))
  end

  def sign_out(resource)
    logout(warden_scope(resource))
  end

  private

  def warden_scope(resource)
    resource.class.name.underscore.to_sym
  end
end
```

それから spec/rails_helper.rb を開き、先ほど作成したヘルパーメソッドをリクエストスペックで使えるようにします。<br>

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
Dir[Rails.root.join('spec', 'support', '**', '*.rb')].sort.each { |f| require f }

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

  # コントローラスペックで Devise のテストヘルパーを使用する
  config.include Devise::Test::ControllerHelpers, type: :controller
  config.include RequestSpecHelper, type: :request ## 追加
end
```

bundle exec rspec spec/requests コマンドを実行して新しく追加したテストを動かしてみてください。<br>
いつものようにエクスペクテーションをいろいろ変えて遊んでみましょう。テストを失敗させ、それからまた元に戻してみましょう。<br>
さあこれでみなさんはコントローラスペックとリクエストスペックの両方を書けるようになりました。<br>
では、どちらのテストを書くべきでしょうか?<br>
第5章でもお話ししたとおり、私はコントローラスペックよりも統合スペック(システムスペックとリクエストスペック)を強くお勧めします。<br>
なぜなら Rails におけるコントローラスペックは重要性が低下し、かわりにより高いレベルのテストの重要性が上がってきているためです。<br>
こうしたテストの方がア プリケーションのより広い範囲をテストすることができます。<br>
とはいえ、コントローラレベルのテストを書く方法は人によってさまざまです。なので、とりあえずどちらのテストも書けるように練習しておいた方が良いと思います。<br>
実際、みなさんはこれでどちらのテストも書けるようになりました。<br>

+ `$ bundle exec rspec spec/requests`を実行(パスする)<br>

## まとめ

Railsで作成した API をテストすることの重要性は徐々に上がってきています。<br>
なぜなら、最近ではアプリケーション同士がやりとりする機会が増えてきているからです。テストスイートを自分の API のクライアントだと考えるようにしてください。<br>
すでにお伝えしたとおり、この章は API の作り方を教える章ではありません。ですが、テストを利用して他のクライアントに公開しているインターフェースを改善することもできます。<br>

さて、これで典型的な Rails アプリケーションの全レイヤーをテストしました。<br>
しかし、これまで書いたテストコードではコードが重複している部分もあります。<br>
次の章ではこうしたコードの重複を無くしていきます。<br>
それだけでなく、テストを書くときに、あえて重複を残したままにするケースについても見ていきます<br>

## 演習問題

• サンプルアプリケーションに別の API エンドポイントを追加してください。<br>
既存のプロジェクトAPIに追加してもいいですし、タスクやその他の機能にアクセスするAPIを追加しても構いません。<br>
それから、そのAPIのテストを書いてください(可能ならテスト から先に書き始めてみましょう!)。<br>

• 自分のアプリケーションですでにコントローラスペックを書いている場合、そのテストをリクエストスペックに移行する方法を考えてみてください。<br>
リクエストスペックではどんな違いが出てくるでしょうか?<br>
