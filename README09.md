# 6. システムスペックで UI をテストする p105〜

```
現時点で私たちはプロジェクト管理ソフトウェアのテストをかなりたくさん作ってきました。
RSpecのインストールと設定を終えたあと、モデルとコントローラの単体テストを作りました。
テストデータを生成するためにファクトリも使いました。さて今度はこれら全部を一緒に使って統合テストを作ります。
言い換えるなら、モデルとコントローラが他のモデルやコントローラとうまく一緒に動作することを確認します。

このようなテストをRSpecではシステムスペック(system specs)と呼んでいます。システムスペックは受入テスト 、または 統合テストと呼ばれることもあります。この種のテストでは開発したソフトウェア全体が一つのシステムとして期待どおりに動くことを検証します。
システムスペックのコツを一度つかめば、Rails アプリケーション内の様々な機能をテストできるようになります。
またシステムスペックはユーザーから上がってきたバグレポートを再現させる際にも利用できます。
嬉しいことに、あなたは堅牢なシステムスペックを書くために必要な知識をほとんど全部身につけています。

システムスペックの構造はモデルやコントローラとよく似ているからです。FactoryBotを使ってテストデータを生成することもできます。
この章では Capybara を紹介します。Capybara は大変便利な Ruby ライブラリで、システムスペックのステップを定義したり、アプリケーションの実際の使われ方をシミュレートしたりするのに役立ちます。

本章ではシステムスペックの基礎を説明します。
• まず最初に、システムスペックをいつ、そしてなぜ書くのかを他の選択肢と比較しなが ら考えてみます。
• 次に、統合テストで必要になる追加ライブラリについて説明します。
• それからシステムスペックの基礎を見ていきます。
• そのあと、もう少し高度なアプローチ、すなわち JavaScript が必要になる場合のテスト
に取り組みます。
• 最後に、システムスペックのベストプラクティスを少し考えて本章を締めくくります。
```

## なぜシステムスペックなのか？

```
私たちは大変⻑い時間をかけてコントローラのテストに取りくんできました。にもかかわらず、なぜ別のレイヤーをテストしようとするのでしょうか?
それはなぜなら、コントローラのテストは比較的シンプルな単体テストであり、結局アプリケーションのごく一部をテストしているに過ぎないからです。
システムスペックはより広い部分をカバーし、実際のユーザーがあなたのコードとどのようにやりとりするのかを表現します。
言い換えるなら、システムスペックではたくさんの異なる部品が統合されて、一つのアプリケーションになっていることをテストします。

Rails 5.1以降の Rails ではシステムテスト(system test)という名前で、このタイプのテストがセットアップ時にデフォルトでサポートされています。
内容的にはこの章で説明するシステムスペックとほとんど同じです。
この章では Rails 標準の Minitest ではなく RSpec を使うため、のちほど設定を変更します。
```

## システムスペックで使用する gem

```
前述のとおり、ここではブラウザの操作をシミュレートするために Capybara を使います。
Capybaraを使うとリンクをクリックしたり、Webフォームを入力したり、画面の表示を検証したりすることができます。
Rails 5.1以降のRailsであれば Capybaraはすでにインストールされています。
なぜなら Capybara はシステムテストでも利用されるからです。
念のため Gemfileを開き、テスト環境に Capybara が追加されていることを確認してください。
```

## システムスペックの基本

```
Capybara を使うと高レベルなテストを書くことができます。
Capybara では click_link や fill_in 、visit といった理解しやすいメソッドが提供されていて、アプリケーションで必要な機能のシナリオを書くことができるのです。
今から実際にやってみましょう。
ここではジェネレータを使って新しいテストファイルを作成します。
最初に rails generate rspec:system projects とコマンドラインに入力してください。
作成されたファイルは次の ようになっています。
```

+ `$ rails g rspec:system projects`を実行<br>

+ `rspec/system/projects_spec.rb`を編集<br>

```rb:projects_spec.rb
require 'rails_helper'

RSpec.describe "Projects", type: :system do
  before do
    driven_by(:rack_test)
  end

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
      expect(page).to have_content "Owner #{user.name}"
    }.to change(user.projects, :count).by(1)
  end
end
```

+ `$ bundle exec rspec`を実行してパスすればOK<br>

```
このスペックのステップを順番に見ていくと、最初に新しいテストユーザーを作成し、次にログイン画面からそのユーザーでログインしています。
それからアプリケーションの利用者が使うものとまったく同じ Web フォームを使って新しいプロジェクトを作成しています。
これはシステムスペックとコントローラスペックの重要な違いです。

コントローラスペックではユーザーインターフェースを無視して、パラメータを直接コントローラのメソッドに送信します。
この場合のメソッドは複数のコントローラと複数のアクションになります。

具体的には home#index 、sessions#new 、projects#index 、projects#new 、それに projects#create です。
しかし、結果は同じになります。新しいプロジェクトが作成され、アプリケーションはそのプロジェクト画面へリダイレクトし、処理の成功を伝えるフラッシュメッセージが表示され、ユーザーはプロジェクトのオーナーとして表示されます。
ひとつの スペックで全部できています!

ここで expect{} の部分に少し着目してください。この中ではブラウザ上でテストしたいステップを明示的に記述し、それから、結果の表示が期待どおりになっていることを検証しています。
ここで使われているのは Capybara の DSL です。自然な英文になっているかというとそうでもありませんが、それでも理解はしやすいはずです。

expect{} ブロックの最後では change マッチャを使って最後の重要なテスト、つまり「ユ ーザーがオーナーになっているプロジェクトが本当に増えたかどうか」を検証しています。
```

＊ click_button を使うと、起動されたアクションが完了する前に次の処理へ移ってしまうことがあります。<br>
そこで、click_button を実行した expect{} の内部で最低でも1個以上のエクスペクテーションを実行し、処理の完了を待つようにするのが良いでしょう。<br>
このサンプルコードでもそのようにしています。<br>

```
scenario は it と同様に example の起点を表しています。
scenario の代わりに、標準的な it 構文に置き換えることもできます。
RSpec のドキュメントにあるシステムスペックの説明では it が使われています。以下は変更後のコード例です。
```

+ 例です(編集はしない)

```rb:projects_spec.rb
require 'rails_helper'

RSpec.describe "Project", type: :system do
  # before ブロックの記述は省略 ...

  it "user creates a new project" do
    # exampleの中身
end
```

```
個人的には、このまま(訳注: it "user creates ... のままにしておくこと)だと英文として不完全で読みづらいので、説明の文言を it "creates a new project as a user" のように変更することも検討すべきだと思います。
また、もうひとつの代替案として、describe や context ブロックを使い、ブロック内のテストが as a user (ユーザーとして)であることを明示するのも良いかもしれません。
このようにいくつかの選択肢がありますが、本書では scenario を使うことにします。
```

```
さて、最後のポイントを今から話します。システムスペックでは一つの example、もしく は一つのシナリオで複数のエクスペクテーションを書くのは全く問題ありません。一般的に システムスペックの実行には時間がかかります。これまでに書いてきたモデルやコントロー ラの小さな example に比べると、セットアップや実行にずっと時間がかかります。また、テ ストの途中でエクスペクテーションを追加するのも問題ありません。たとえば、一つ前のス ペックの中で、ログインの成功がフラッシュメッセージで通知されることを検証しても良い わけです。しかし本来、こういうエクスペクテーションを書くのであれば、ログイン機能の 細かい動きを検証するために専用のシステムスペックを用意する方が望ましいでしょう。
```

## CapybaraのDSL

```
先ほど作ったテストでは読者のみなさんはすでにお馴染みであろう RSpec の構文(expect )と、ブラウザ上の操作をシミュレートする Capybara のメソッドを組み合わせて使いました。
このテストではページを訪問し(visit )、ハイパーリンクにアクセスするためにリンクをクリックし(click_link )、フォームの入力項目に値を入力し(fill_in と with )、ボタン をクリックして入力値を処理しました(click_button )。
ですが、Capybara でできることはもっとたくさんあります。
以下のサンプルコードは Capybara の DSL が提供しているその他のメソッドの使用例です。
```

```rb:sample.rb
# 全種類の HTML 要素を扱う
scenario "works with all kinds of HTML elements" do
  # ページを開く
  visit "/fake/page"
  # リンクまたはボタンのラベルをクリックする
  click_on "A link or button label"
  # チェックボックスのラベルをチェックする
  check "A checkbox label"
  # チェックボックスのラベルのチェックを外す
  uncheck "A checkbox label"
  # ラジオボタンのラベルを選択する
  choose "A radio button label"
  # セレクトメニューからオプションを選択する
  select "An option", from: "A select menu"
  # ファイルアップロードのラベルでファイルを添付する
  attach_file "A file upload label", "/some/file/in/my/test/suite.gif"
  # 指定した CSS に一致する要素が存在することを検証する
  expect(page).to have_css "h2#subheading"
  # 指定したセレクタに一致する要素が存在することを検証する
  expect(page).to have_selector "ul li"
  # 現在のパスが指定されたパスであることを検証する
  expect(page).to have_current_path "/projects/new"
end
```

```
セレクタの スコープ を制限することもできます。
その場合は Capybara の within を使ってページの一部分に含まれる要素を操作します。
```

```html:sample.html.erb
<div id="node">
  <a href="http://nodejs.org">click here!</a>
</div>
<div id="rails">
  <a href="http://rubyonrails.org">click here!</a>
</div>
```

上のようなHTMLでは次のようにしてアクセスしたい `click here!` のリンクを選択できます。<br>

```rb:sample.rb
within "#rails" do
  click_link "click here!"
end
```

もしテスト内で指定したセレクタに合致する要素が複数見つかり、Capybara にあいまいだ (ambiguous)と怒られたら、within ブロックで要素を内包し、あいまいさをなくしてみてください。<br>

また、Capybara にはさまざまな find メソッドもあります。これを使うと値を指定して特定の要素を取り出すこともできます。<br>
たとえば次のような感じです。<br>

```rb:sample.rb
language = find_field("Progaraming language").value
expect(language).to eq "Ruby"

finc("#fine_print").find("#disclaimer").click
find_button("Publish").click
```

ここで紹介した Capybara のメソッドは、私が普段よく使うメソッドです。<br>
ですが、テスト内で使用できる Capybara の全機能を紹介したわけではありません。<br>
全容を知りたい場合は`Capybara DSL のドキュメント`を参照してください。<br>
また、このドキュメントを便利なリファレンスとして手元に置いておくのもいいでしょう。<br>
これ以降の章でも、まだ紹介していない機能をもうちょっと使っていきます。<br>

## システムスペックをデバッグする

Capybara のコンソール出力を読めば、どこでテストが失敗したのか調査することができます。<br>
ですが、それだけでは原因の一部分しかわからないことがときどきあります。<br>
たとえば 次のシステムスペックを見てください。この場合、ユーザーはログインしていないのでテストは失敗します。<br>

+ `spec/system/projects_spec.rb`を編集<br>

```rb:projects_spec.rb
require 'rails_helper'

RSpec.describe "Projects", type: :system do
  before do
    driven_by(:rack_test)
  end

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
      expect(page).to have_content "Owner" #{user.name}"
    }.to change(user.projects, :count).by(1)
  end

  # 追加
  # ゲストがプロジェクトを追加する
  scenario "guest adds a project" do
    visit projects_path
    click_link "New Project"
  end
  # ここまで
end
```

+ `$ bundle exec rspec spec/system`を実行(失敗する)<br>

```:terminal
Failures:

  1) Projects guest adds a project
     Failure/Error: click_link "New Project"

     Capybara::ElementNotFound:
       Unable to find link "New Project"
```

driven_by メソッドで :rack_test を指定した場合、Capybara は ヘッドレス ブラウザ(訳 注: UI を持たないブラウザ)を使ってテストを実行するため、処理ステップを一つずつ目で確認することはできません。<br>
ですが、Rails がブラウザに返した HTML を見ることはできます。<br>
次のように save_and_open_page をテストが失敗する場所の直前に挟み込んでみてください。<br>

+ `spec/system/projects_spec.rb`を編集<br>

```rb:projects_spec.rb
require 'rails_helper'

RSpec.describe "Projects", type: :system do
  before do
    driven_by(:rack_test)
  end

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
      expect(page).to have_content "Owner" #{user.name}"
    }.to change(user.projects, :count).by(1)
  end

  # ゲストがプロジェクトを追加する 確認後削除
  scenario "guest adds a project" do
    visit projects_path
    save_and_open_page # 追加
    click_link "New Project"
  end
end
```

この状態でテストを実行すると、同じ理由でテストは失敗するものの、新しい情報が手に入ります。<br>

+ `$ bundle exec rspec spec/system`を実行<br>

* 日本語版のサンプルアプリケーションでは後述する Launchy gem を予めインストールしてあるので、自動的にブラウザが立ち上がります。

なるほど!ボタンにアクセスできないのは、ユーザーがログインしていなかったからですね。<br>
プロジェクト一覧画面ではなく、ログイン画面にリダイレクトされていたわけです。<br>

この機能はとても便利ですが、毎回手作業でファイルを開く必要はありません。<br>
コンソール出力にも書いてあるとおり、Launchy gem(サンプルではインストール済み) をインストールすれば自動的に開くようになります。<br>
Gemfile にこの gem を追加し、bundle install を実行してください。<br>

こうすれば save_and_open_page をスペック内で呼びだしたときに、Launchy が保存された HTML を自動的に開いてくれます。<br>
ブラウザを起動する必要がない場合や、ブラウザを起動できないコンテナ環境などでは 代わりに save_page メソッドを使ってください。<br>
このメソッドを使うと HTML ファイルが tmp/capybara に保存されます。ブラウザは起動しません。<br>

save_and_open_page や save_page はデバッグ用のメソッドです。システムスペックがパスするようになったら、それ以上のチェックは不要です。<br>
なので、不要になったタイミングでこのメソッド呼び出しは全部削除してください。<br>
削除しないままバージョン管理ツールにコミットしてしまわないよう注意しましょう。

## JavaScriptを使った操作をテストする p113〜

というわけで、私たちはシステムスペックを使ってプロジェクトを追加する UI が期待どおりに動作することを検証しました。<br>
ここで紹介した方法を使えば、Web 画面上の操作の大半をテストすることができます。<br>
ここまで Capybara はシンプルなブラウザシミュレータ(つまりドライバ)を使って、テストに書かれたタスクを実行してきました。<br>

このドライバは Rack::Test というドライバで、速くて信頼性が高いのですが、JavaScript の実行はサポートしていません。<br>
本書のサンプルアプリケーションでは1箇所だけ JavaScript に依存する機能があります。<br>

それはタスクの隣にあるチェックボックスをクリックするとそのタスクが完了状態になる、という機能です。<br>
新しいスペックを書いてこの機能をテストしてみましょう。<br>
システムスペックのジェネレータを使うか、もしくは自分の手で次のような spec/system/tasks_spec.rb という 新しいファイルを追加してください。<br>

+ `$ rails g rspec:system tasks`を実行<br>

+ `spec/system/tasks_spec.rb`を編集<br>

```rb:tasks_spec.rb
require 'rails_helper'

RSpec.describe "Tasks", type: :system do
  # ユーザーがタスクの状態を切り替える
  scenario 'user toggles a task', js: true do
    user = FactoryBot.create(:user)
    project = FactoryBot.create(:project,
      name: "RSpec tutorial",
      owner: user)
    task = project.tasks.create!(name: "Finish RSpec tutorial")

    visit root_path
    click_link "Sign in"
    fill_in "Email", with: user.email
    fill_in "Password", with: user.password
    click_button "Log in"

    click_link "RSpec tutorial"
    check "Finish RSpec tutorial"

    expect(page).to have_css "label#task_#{task.id}.completed"
    expect(task.reload).to be_completed

    uncheck "Finish RSpec tutorial"

    expect(page).to_not have_css "label#{task.id}.completed"
    expect(task.reload).to_not be_completed
  end
end
```

最初に説明した projects_spec.rb では before ブロックで :rack_test というドライバを 指定していましたが、今回はこのあと別の方法でドライバを指定するため driven_by メソッドの記述はなくしています。<br>
加えて、ここでは js: true というオプション(タグ)を渡しています。<br>
このようにして、指定したテストに対して JavaScript が使えるドライバを使うようにタグを付けておきます。<br>
このサンプルアプリケーションでは selenium-webdriver gem を使います。この gem は Rails 5.1以降の Rails にはデフォルトでインストールされていて、Capybara でもデフォルトの JavaScript ドライバになっています。<br>

使用するドライバは driven_by メソッドを使ってテストごとに変更することができます。 <br>
ですが、私は可能な限りシステム全体の共通設定とします。今からその共通設定を追加していきましょう。<br>
rails_helper.rb ファイルはきれいな状態を保っておきたいので、今回は独立したファイルに新しい設定を書くことにします。<br>
RSpec はこのようなニーズをサポートしてくれているので、 簡単な方法で有効化することができます。<br>
spec/rails_helper.rb 内にある以下の行のコメントを外してください。<br>

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
Dir[Rails.root.join('spec', 'support', '**', '*.rb')].sort.each { |f| require f } # コメントアウトを解除する

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
end
```

こうすると RSpec 関連の設定ファイルを spec/support ディレクトリに配置することができます。<br>
Devise 用の設定を追加したときのように、spec/rails_helper.rb 内に直接設定を書き込まなくても済むのです。<br>
それでは spec/support/capybara.rb という新しいファイルを作成し、次のような設定を追加しましょう。<br>

+ `$ mkdir spec/support && touch $_/capybara.rb`を実行<br>

+ `spec/support/capybara.rb`を編集<br>

```rb:capybara.rb
RSpec.configure do |config|
  config.before(:each, type: :system) do
    driven_by :rack_test
  end

  config.before(:each, type: :system, js: true) do
    driven_by :selenium_chrome
  end
end
```

ここではブラウザを使った基本的なテストでは高速な Rack::Test ドライバを使い、より複雑なブラウザ操作が必要な場合は JavaScript が実行可能なドライバ(ここでは selenium- webdriver と Chrome)を設定するようにしています。<br>
どちらのドライバを使用するのかはタグで識別します。<br>
デフォルトでは Rack::Test ドライバを使いますが、js: true のタグが付 いているテストに限り、selenium-webdriver と Chrome を使う設定になっています。<br>

このほかに Chrome とやりとりするインターフェースになる ChromeDriver が必要になります。<br>
ChromeDriver 自体は Ruby の gem ではありませんが、簡単にインストールできる Webdrivers gem があります。<br>
この gem は Rails 6.0以降の Rails なら標準でインストールされています。<br>
Webdrivers gem がインストールされていなければ、次のように Gemfile に追加して bundle install を実行してください。<br>

＊ 一点補足しておくと、Webdrivers はライブラリの依存関係から selenium-webdriver gem も一緒にインストールしてくれるため、Gemfile に selenium-webdriver を明示的に書かなくても済むようになります。<br>

+ `$ bundle exec rspec spec/system/tasks_spec.rb`を実行<br>

設定がうまくいっていれば、Chrome のウィンドウが新しく立ち上がります(ただし、現在 開いている他のウィンドウのうしろに隠れているかもしれません)。<br>
ウィンドウ内ではサンプルアプリケーションが開かれ、目に見えない指がリンクをクリックし、フォームの入力項目を入力し、タスクの完了状態と未完了状態を切り替えます。素晴らしい!<br>

テストはパスしましたが、このテストの遅さに注目してください!<br>
これは JavaScript を実行するテストと、Selenium を使うテストのデメリットです。<br>
一方で、セットアップは比較的簡単ですし、私たち自身が自分の手で操作する時間に比べたら、こちらの方がまだ速いです。<br>
ですが、もし一つのテストを実行するのに(私のマシンで)8秒以上かかるのであれば、この先 JavaScript を使う機能とそれに対応するテストを追加していったら、どれくらいの時間がかかるでしょうか?<br>

JavaScript ドライバはだんだん速くなっているので、そのうちいつか Rack::Test と同等のスピードで実行できるようになるかもしれません。<br>
ですが、それまでは必要なときにだけ、テスト上で JavaScript を有効にする方が良い、というのが私からのアドバイスです。<br>
最後の仕上げとして、私たちのシステムスペックではデフォルトで Rack::Test ドライバ を使うようになったため、projects_spec.rb の before ブロックは削除しても大丈夫です。 <br>
projects_spec.rb から before ブロックを削除すると次のようになります。<br>

+ `spec/system/projects_spec.rb`を編集 p117〜<br>

```rb:projects_spec.rb
require 'rails_helper'

RSpec.describe "Projects", type: :system do
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
      expect(page).to have_content "Owner" #{user.name}"
    }.to change(user.projects, :count).by(1)
  end
end
```

+ `$ bundle exec rspec spec/system/projects_spec.rb`を実行<br>

before ブロックを削除したあともこれまでと同様に Chrome が起動することなくテストが完了すれば OK です。<br>
