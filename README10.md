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
