# スペックをDRYに保つ p133

みなさんがここまでに学んだ知識を使って自分のアプリケーションにテストを書いていけば、しっかりしたテストスイートがきっとできあがるはずです。<br>
しかし、コードはたくさん重複しています。いわば、[Don’t Repeat Yourself](http://wiki.c2.com/?DontRepeatYourself) (DRY) 原則を破っている状態です。<br>
アプリケーションコードと同様、テストスイートをきれいにすることも検討しましょう。<br>
この章では RSpec が提供しているツールを使い、複数のテストをまたがってコードを共有する方法を説明します。<br>
また、どのくらい DRY になると DRY すぎるのかについても説明します。<br>

この章で説明する内容は以下のとおりです。<br>

• ワークフローをサポートモジュールに切り出す<br>

• テスト内でインスタンス変数を再利用するかわりにletを使う<br>

• shared_contextに共通のセットアップを移動する<br>

• RSpec と rspec-rails で提供されているマッチャに加えて、カスタムマッチャを作成する<br>

• エクスペクテーションを集約して、複数のスペックをひとつにする<br>

• テストの何を抽象化し、何をそのまま残すか判断する<br>

## サポートモジュール

ここまでに作ったシステムスペックをあらためて見てみましょう。<br>
今のところ、たった二つのスペックしか書いていませんが、どちらのテストにもユーザーがアプリケーションにログインするステップが含まれています。<br>

```rb:sample.rb
visit root_path
click_link "Sign in"
fill_in "Email", with: user.email
fill_in "Password", with: user.password
click_button "Log in"
```

もしログイン処理が変わったらどうなるでしょうか? たとえば、ボタンのラベルが変わるような場合です。<br>
こんな単純な変更であっても、いちいち全部のテストコードを変更しなけ ればいけません。<br>
この重複をなくすシンプルな方法は `サポートモジュール` を使うことです。<br>
ではコードを新しいモジュールに切り出してみましょう。spec/support ディレクトリに login_support.rb という名前のファイルを追加し、次のようなコードを書いてください。<br>

+ `$ touch spec/support/login_support.rb`を実行<br>

+ `spec/support/login_support.rb`を編集<br>

```rb:login_support.rb
module LoginSupport
  def sign_in_as(user)
    visit root_path
    click_link "Sign in"
    fill_in "Email", with: user.email
    fill_in "Password", with: user.password
    click_button "Log in"
  end
end

RSpec.configure do |config|
  config.include LoginSupport
end
```

このモジュールにはメソッドがひとつ含まれます。コードの内容は元のテストで重複していたログインのステップです。<br>
モジュールの定義のあとには、RSpec の設定が続きます。<br>
ここでは RSpec.configure を使って新しく作ったモジュールを include しています。<br>
これは必ずしも必要ではありません。テスト毎に明示的にサポートモジュールを include する方法もあります。<br>
たとえば次のような感じです。<br>

p134〜<br>
+ `spec/system/projects_spec.rb`を編集<br>

```rb:projects_spec.rb
require 'rails_helper'

RSpec.describe "Projects", type: :system do
  include LoginSupport # 追加

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

さあ、これでログインのステップが重複している二つのスペックをシンプルにすることができます。<br>
また、この先で同じステップが必要になるスペックでも、このヘルパーメソッドを使うことができます。<br>
たとえば、プロジェクトのシステムスペックは次のように書き換えられます。<br>

+ `spec/system/projects_spec.rb`を編集<br>

```rb:projects_spec.rb
require 'rails_helper'

RSpec.describe "Projects", type: :system do
  include LoginSupport

  # ユーザーは新しいプロジェクトを作成する
  scenario "user creates a new project" do
    user = FactoryBot.create(:user)
    sign_in_as user # 追加

    # 編集
    expect {
      click_link "New Project"
      fill_in "Name", with: "Test Project"
      fill_in "Description", with: "Trying out Capybara"
      click_button "Create Project"
    }.to change(user.projects, :count).by(1)

    expect(page).to have_content "Project was successfully created"
    expect(page).to have_content "Test Project"
    expect(page).to have_content "Owner: #{user.name}"
    # ここまで
  end
end
```

+ `$ bundle exec rspec spec/system`を実行するとパスする<br>

共通のワークフローをサポートモジュールに切り出す方法は、コードの重複を減らすお気に入りの方法の一つで、とくに、システムスペックでよく使います。<br>
モジュール内のメソッド名は、コードを読んだときに目的がぱっとわかるような名前にしてください。<br>
もしメソッドの処理を理解するために、いちいちファイルを切り替える必要があるのなら、それはかえってテストを不便にしてしまっています。<br>

ここで適用したような変更は過去にもやっています。それが何だかわかりますか? <br>
Deviseはログインのステップを完全に省略できるヘルパーメソッドを提供しています。これを使えば、特定のユーザーに対して即座にセッションを作成できます。<br>
これを使えば UI の操作をシミュレートするよりずっと速いですし、ユーザーがログイン済みになっていることがテストを実行する上での重要な要件になっている場合は大変便利です。<br>
別の見方をすれば、ここでテストしたいのはプロジェクトの機能であって、ユーザーの機能やログインの機能ではない、ということもできます。<br>

これを有効化するために rails_helper.rb を開き、他の Devise の設定に続けて次のようなコードを追加してください(訳注: Devise::Test::IntegrationHelpers の行を追加します)。<br>

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
  config.include RequestSpecHelper, type: :request
  config.include Devise::Test::IntegrationHelpers, type: :system ## 追加
end
```

さあ、これで今回独自に作った sign_in_as メソッドを呼び出す部分は、Devise の sign_in ヘルパーで置き換えることができます。<br>

p136〜
+ `spec/system/projects_spec.rb`を編集<br>

```rb:projects_spec.rb
require 'rails_helper'

RSpec.describe "Projects", type: :system do
  include LoginSupport

  # ユーザーは新しいプロジェクトを作成する
  scenario "user creates a new project" do
    user = FactoryBot.create(:user)
    sign_in user # 編集

    expect {
      click_link "New Project"
      fill_in "Name", with: "Test Project"
      fill_in "Description", with: "Trying out Capybara"
      click_button "Create Project"
    }.to change(user.projects, :count).by(1)

    expect(page).to have_content "Project was successfully created"
    expect(page).to have_content "Test Project"
    expect(page).to have_content "Owner: #{user.name}"
  end
end
```

+ `$ bundle exec rspec spec/system`を実行<br>

```:terminal
Projects
  user creates a new project (FAILED - 1)

Tasks
  user toggles a task

Failures:

  1) Projects user creates a new project
     Failure/Error: click_link "New Project"

     Capybara::ElementNotFound:
       Unable to find link "New Project"
```

いったい何が起こったかわかりますか?<br>
もしわからなければ、save_and_open_page メソッドを各スペックで失敗している click_link メソッドの直前で呼び出してください。<br>
これはどうやら独自に作ったログインヘルパーと、ヘルパーに切り出す前の元のステップでは、ログイン後にユーザーのホームページに遷移する副作用があったようです。<br>
しかし、Deviseのヘルパーメソッドではセッションを作成するだけなので、どこからワークフローを開始するのかテスト内で明示的に記述しなければなりません(訳注: sign_in user に続けて、visit root_path を追加します)。<br>

+ `spec/system/projects_spec.rb`を編集<br>

```rb:projects_spec.rb
require 'rails_helper'

RSpec.describe "Projects", type: :system do
  include LoginSupport

  # ユーザーは新しいプロジェクトを作成する
  scenario "user creates a new project" do
    user = FactoryBot.create(:user)
    sign_in user

    visit root_path # 追加

    expect {
      click_link "New Project"
      fill_in "Name", with: "Test Project"
      fill_in "Description", with: "Trying out Capybara"
      click_button "Create Project"
    }.to change(user.projects, :count).by(1)

    expect(page).to have_content "Project was successfully created"
    expect(page).to have_content "Test Project"
    expect(page).to have_content "Owner: #{user.name}"
  end
end
```

+ `$ bundle exec rspec spec/system`を実行(パスする)<br>

このあとも同じようなコードを書いていく点に注意してください。<br>
独自のサポートメソッドのような仕組みを利用する場合、テスト内で明示的に次のステップを記述するのは基本的によい考えです。<br>
そうすることで、ワークフローが文書化されます。繰り返しになりますが、今回のサンプルコードではユーザーがログイン済みになっていることは、あくまでセットアップ上の要件にすぎません。<br>
ログインのステップ自体はテスト上の重要な機能になっているわけではない、という点を押さえておきましょう。<br>
今回使った Devise のヘルパーメソッドはこのあとのテストでも使っていきます。<br>

## let で遅延読み込みする p138〜

私たちはここまでに before ブロックを使ってテストを DRY にしてきました。<br>
before ブロックを使うと describe や context ブロックの内部で、各テストの実行前に共通のインスタンス変数をセットアップできます。<br>

この方法も悪くはないのですが、まだ解決できていない問題が二つあります。<br>
第一に、before の中に書いたコードは describe や context の内部に書いたテストを実行するたびに毎回実行されます。<br>
これはテストに予期しない影響を及ぼす恐れがあります。また、そうした問題が起きない場合でも、使う必要のないデータを作成してテストを遅くする原因になることもあります。<br>

第二に、要件が増えるにつれてテストの 可読性を悪くします。<br>
こうした問題に対処するため、RSpec は let というメソッドを提供しています。let は呼ばれたときに初めてデータを読み込む、遅延読み込みを実現するメソッドです。<br>
let は before ブロックの 外部で呼ばれるため、セットアップに必要なテストの構造を減らすこともできます。<br>
let の使い方を説明するために、タスクモデルのモデルスペックを作成してみましょう。このスペックはまだ作成していませんでした。<br>
rspec:model ジェネレータを使うか、spec/models に自分でファイルを作るかして、スペックファイルを作成してください。(ジェネレータを 使う場合は、タスク用の新しいファクトリも作られるはずです。)それから次のようなコードを書いてください。<br>

p138〜<br>
+ `$ bin/rails g rspec:model task`を実行<br>

+ `spec/models/task_spec.rb`を編集<br>

```rb:task_spec.rb
require 'rails_helper'

RSpec.describe Task, type: :model do
  let(:project) { FactoryBot.create(:project) }

  # プロジェクトと名前があれば有効な状態であること
  it "is valid with a project and name" do
    task = Task.new(
      project: project,
      name: "Test task",
    )
    expect(task).to be_valid
  end

  # プロジェクトがなければ無効な状態であること
  it "is invalid without a project" do
    task = Task.new(project: nil)
    task.valid?
    expect(task.errors[:project]).to include("must exist")
  end

  # 名前がなければ無効な状態あること
  it "is invalid witout a name" do
    task = Task.new(name: nil)
    task.valid?
    expect(task.errors[:name]).to include("can't be blank")
  end
end
```

今回は4行目にある let を使って必要となるプロジェクトを作成しています。<br>
しかし、プロジェクトが作成されるのはプロジェクトが必要になるテストだけです。最初のテストはプロジェクトを作成します。<br>
なぜなら9行目で project が呼ばれるからです。project は4行目の let で作られた値を呼び出します。let は新しいプロジェクトを作成します。<br>
テストの実行が終わると、12行目以降のテストではプロジェクトが取り除かれます。他の2つのテストではプロジェクトを使いません。<br>
実際、テストは「プロジェクトがなければ無効な状態であること」というテストなので、本当にプロジェクトがいらないのです。<br>
なので、この二つのテストではプロジェクトはまったく作成されません。<br>

let を使う場合はちょっとした違いがあります。before ブロックでテストデータをセットアップする際は、インスタンス変数に格納していたことを覚えていますか?<br>
let を使ったデータに関してはこれが当てはまりません。なので、9行目を見てみると、@project ではなく、project でデータを呼び出しています。<br>
let は必要に応じてデータを作成するので、注意しないとトラブルの原因になることもあります。<br>
たとえば、メモ(Note)のモデルスペックをファクトリと let を使ってリファクタリングしてみましょう。<br>

+ `spec/models/note_spec.rb`を編集<br>

```rb:note_spec.rb
require 'rails_helper'

RSpec.describe Note, type: :model do
  let(:user) { FactoryBot.create(:user) }
  let(:project) { FactoryBot.create(:project, owner: user) }

  # ユーザー、プロジェクト、メッセージがあれば有効な状態であること
  it "is valid a user, project, and message" do
    note = Note.new(
      message: "This is a same note.",
      user: user,
      project: project
    )
    expect(note).to be_valid
  end

  # メッセージがなければ無効な状態であること
  it "is invalid without a message" do
    note = Note.new(message: nil)
    note.valid?
    expect(note.errors[:message]).to include("can't be blank")
  end

  # 文字列に一致するメッセージを検索する
  describe "search message for a term" do
    let(:note1) { FactoryBot.create(:note,
    project: project,
    user: user,
    message: "This is the first note.",
    )
  }

    let(:note2) {
      FactoryBot.create(:note,
        project: project,
        user: user,
        message: "This is the second note.",
      )
    }

    let(:note3) {
      FactoryBot.create(:note,
        project: project,
        user: user,
        message: "First, preheat the oven.",
      )
    }

    # 一致するデータが見つかるとき
    context "when a match is found" do
      # 文字列に一致するメモを返すこと
      it "returns notes that match the search term" do
        expect(Note.search("first")).to include(note1, note3)
      end
    end

    # 一致するデータが1件も見つからないとき
    context "when no match is found" do
      # 空のコレクションを返すこと
      it "returns an empty collection" do
        expect(Note.search("message")).to be_empty
      end
    end
  end
end
```

+ `$ bundle exec rspec`を実行する(パスする)`<br>

コードがちょっときれいになりましたね。なぜなら before ブロックでインスタンス変数をセットアップする必要がなくなったからです。<br>
実行してみると一発でテストがパスします!<br>
しかし一つ問題があります。ためしに returns an empty collection のテストで次の一行(訳 注: expect(Note.count).to eq 3 )を追加してみましょう。<br>

+ `spec/models/note_spec.rb`を編集<br>

```rb:note_spec.rb
require 'rails_helper'

RSpec.describe Note, type: :model do
  let(:user) { FactoryBot.create(:user) }
  let(:project) { FactoryBot.create(:project, owner: user) }

  # ユーザー、プロジェクト、メッセージがあれば有効な状態であること
  it "is valid a user, project, and message" do
    note = Note.new(
      message: "This is a same note.",
      user: user,
      project: project
    )
    expect(note).to be_valid
  end

  # メッセージがなければ無効な状態であること
  it "is invalid without a message" do
    note = Note.new(message: nil)
    note.valid?
    expect(note.errors[:message]).to include("can't be blank")
  end

  # 文字列に一致するメッセージを検索する
  describe "search message for a term" do
    let(:note1) { FactoryBot.create(:note,
    project: project,
    user: user,
    message: "This is the first note.",
    )
  }

    let(:note2) {
      FactoryBot.create(:note,
        project: project,
        user: user,
        message: "This is the second note.",
      )
    }

    let(:note3) {
      FactoryBot.create(:note,
        project: project,
        user: user,
        message: "First, preheat the oven.",
      )
    }

    # 一致するデータが見つかるとき
    context "when a match is found" do
      # 文字列に一致するメモを返すこと
      it "returns notes that match the search term" do
        expect(Note.search("first")).to include(note1, note3)
      end
    end

    # 一致するデータが1件も見つからないとき
    context "when no match is found" do
      # 空のコレクションを返すこと
      it "returns an empty collection" do
        expect(Note.search("message")).to be_empty
        expect(Note.count).to eq 3 # 追加
      end
    end
  end
end
```

+ `$ bundle exec rspec`を実行(失敗する)<br>

```:terminal
Failures:

  1) Note search message for a term when no match is found returns an empty collection
     Failure/Error: expect(Note.count).to eq 3

       expected: 3
            got: 0

       (compared using ==)
     # ./spec/models/note_spec.rb:62:in `block (4 levels) in <main>'
```

いったい何が起きてるんでしょうか?このテストでは note1 と note2 と note3 をどれも明示的に呼び出していません。<br>
なので、データが作られず、search メソッドは何もデータがないデータベースに対して検索をかけます。<br>
当然、検索しても何も見つかりません!<br>
この問題は search メソッドを実行する前に let で作ったメモを強制的に読み込むようにハックすれば解決できます。<br>

+ `spec/models/note_spec.rb`を編集 p142〜<br>

```rb:note_spec.rb
require 'rails_helper'

RSpec.describe Note, type: :model do
  let(:user) { FactoryBot.create(:user) }
  let(:project) { FactoryBot.create(:project, owner: user) }

  # ユーザー、プロジェクト、メッセージがあれば有効な状態であること
  it "is valid a user, project, and message" do
    note = Note.new(
      message: "This is a same note.",
      user: user,
      project: project
    )
    expect(note).to be_valid
  end

  # メッセージがなければ無効な状態であること
  it "is invalid without a message" do
    note = Note.new(message: nil)
    note.valid?
    expect(note.errors[:message]).to include("can't be blank")
  end

  # 文字列に一致するメッセージを検索する
  describe "search message for a term" do
    let(:note1) { FactoryBot.create(:note,
    project: project,
    user: user,
    message: "This is the first note.",
    )
  }

    let(:note2) {
      FactoryBot.create(:note,
        project: project,
        user: user,
        message: "This is the second note.",
      )
    }

    let(:note3) {
      FactoryBot.create(:note,
        project: project,
        user: user,
        message: "First, preheat the oven.",
      )
    }

    # 一致するデータが見つかるとき
    context "when a match is found" do
      # 文字列に一致するメモを返すこと
      it "returns notes that match the search term" do
        expect(Note.search("first")).to include(note1, note3)
      end
    end

    # 一致するデータが1件も見つからないとき
    context "when no match is found" do
      # 空のコレクションを返すこと
      it "returns an empty collection" do
        # 追加
        note1
        note2
        note3
        # ここまで
        expect(Note.search("message")).to be_empty
        expect(Note.count).to eq 3
      end
    end
  end
end
```

+ `$ bundle exec rspec`を実行(パスする)<br>

ですが、私に言わせれば、これはまさにハックです。<br>
私たちは読みやすいスペックを書こうと努力しています。新しく追加した行は読みやすくありません。<br>
そこで、このようなハックをするかわりに、let! を使うことにします。let とは異なり、let! は遅延読み込みされません。<br>
let! はブロックを即座に実行します。なので、内部のデータも即座に作成されます。それでは、let を let! に書き換えましょう。<br>

+ `spec/models/note_spec.rb`を編集<br>

```rb:note_spec.rb
require 'rails_helper'

RSpec.describe Note, type: :model do
  let(:user) { FactoryBot.create(:user) }
  let(:project) { FactoryBot.create(:project, owner: user) }

  # ユーザー、プロジェクト、メッセージがあれば有効な状態であること
  it "is valid a user, project, and message" do
    note = Note.new(
      message: "This is a same note.",
      user: user,
      project: project
    )
    expect(note).to be_valid
  end

  # メッセージがなければ無効な状態であること
  it "is invalid without a message" do
    note = Note.new(message: nil)
    note.valid?
    expect(note.errors[:message]).to include("can't be blank")
  end

  # 文字列に一致するメッセージを検索する
  describe "search message for a term" do
    let!(:note1) { FactoryBot.create(:note,
    project: project,
    user: user,
    message: "This is the first note.",
    )
  }

    let!(:note2) {
      FactoryBot.create(:note,
        project: project,
        user: user,
        message: "This is the second note.",
      )
    }

    let!(:note3) {
      FactoryBot.create(:note,
        project: project,
        user: user,
        message: "First, preheat the oven.",
      )
    }

    # 一致するデータが見つかるとき
    context "when a match is found" do
      # 文字列に一致するメモを返すこと
      it "returns notes that match the search term" do
        expect(Note.search("first")).to include(note1, note3)
      end
    end

    # 一致するデータが1件も見つからないとき
    context "when no match is found" do
      # 空のコレクションを返すこと
      it "returns an empty collection" do
        # note1
        # note2
        # note3
        expect(Note.search("message")).to be_empty
        expect(Note.count).to eq 3
      end
    end
  end
end
```

+ `$ bundle exec rspec`を実行(パスする)<br>

これで先ほどの実験はパスしました。しかし let! にまったく問題がないわけでもありません。<br>
まず、この変更でテストデータが遅延読み込みされない元の状態に戻ってきてしまいました。この点は今回大した問題にはなっていません。<br>
テストデータを使う example は、どちらも正しく実行するために3件全部のメモが必要です。とはいえ、多少注意する必要はあ ります。<br>
なぜならすべてのテストが余計なデータを持つことになり、予期しない副作用を引き起こすかもしれないからです。<br>

次に、コードを読む際は let と let! の見分けが付きにくく、うっかり読み間違えてしまう可能性があります。<br>
繰り返しになりますが、私たちは読みやすいテストスイートを作ろうと努力しています。<br>
もし、みなさんがこのわずかな違いを確認するためにコードを読み返すようであれば、before とインスタンス変数に戻すことも検討してください。<br>
別に[テストで必要なデータを直接テスト内でセットアップしてしまっても、なんら問題](https://thoughtbot.com/blog/my-issues-with-let)はないのです。<br>
こうした選択肢をいろいろ試し、あなたとあなたのチームにとって最適な方法を見つけてください。<br>
