## 9章: 速くテストを書き、速いテストを書く p 163〜

プログラミングに関する格言の中で私が気に入っているものの一つは、「まず動かし、次に正しくし、それから速くする([Make it work, make it right, make it fast](http://wiki.c2.com/?MakeItWorkMakeItRightMakeItFast))」です。<br>
最初の方の 章では、とりあえずテストが動くようにしました。それから第8章ではテストをリファクタリングして、コードの重複をなくすテクニックを説明しました。<br>
この章では先ほどの格言で 言うところの速くするパートを説明します。<br>
私は速いという用語を二つの意味で使っています。一つはもちろん、スペックの実行時間です。本書のサンプルアプリケーションとテストスイートは徐々に大きくなってきています。<br>
このまま何もせずに開発を進めれば、テストはうんざりするぐらい遅くなっていくことでしょう。<br>
よって目標としたいのは、RSpec の可読性や堅牢なテストスイートによってもたらされる安心感を損なうことなく、RSpec の実行速度を十分満足できる速さに保つことです。<br>
そして、私が意図する二つ目のスピードは、意味がわかりやすくてきれいなスペックを開発者としていかに素早く作るか、ということです。<br>
 本章ではこの両面を説明します。具体的な内容は以下の通りです。<br>

+ 構文を簡潔かつ、きれいにすることでスペックをより短くする RSpec のオプション<br>
+ みなさんのお気に入りのエディタを活用して、キー入力を減らす方法<br>
+ モックとスタブを使って、潜在的なパフォーマンス上のボトルネックをテストから切り離す方法<br>
+ 遅いスペックを除外するためのタグの使用<br>
+ テスト全体をスピードアップするテクニック<br>

## RSpecの簡潔な構文 p163〜

ここまでに書いてきたスペックのいくつかをもう一度見直してみてください。特に、モデルスペックに注目してみましょう。<br>
私たちはこれまでいくつかのベストプラクティスに従い、テストにはわかりやすいラベルを付け、一つの example に一つのエクスペクテーションを書いてきました。<br>
これらは全部目的があってやってきたことです。ここまで見てきたような明示的なアプローチは、私がテストの書き方を学習したときのアプローチを反映しています。こうしたアプローチを使えば、自分のやっていることが理解しやすいはずです。<br>
しかし、RSpec にはキーストロークを減らしながらこうしたベストプラクティスを実践し続けられるテクニックがあります。<br>

一つ前の章では、テストデータを宣言するオプションとして let があることを説明しました。もう一つのメソッドである subject も同じように呼ばれますが、ユースケースがちょっと異なります。<br>
subject はテストの対象物(subject )を宣言します。そして、そのあとに続く example 内で暗黙的に何度でも再利用することができます。<br>
私たちは第3章からここまでずっと、it を⻑い記法で使い続けてきました。⻑い記法とは すなわち、文字列としてテストの説明を自然言語で記述するアプローチのことです。<br>
ですが、この it はただの Ruby のメソッドです。みなさんもすでに気づいているかもしれませんが、このメソッドはブロックを受け取り、そのブロックの中にはいくつかのテストのステップを含めることができます。<br>
これはつまり、シンプルなテストケースであればテストコードを1行にまとめることができるかもしれないということです!<br>
これを実現するために、ここでは RSpec の is_expected メソッドを使用します。is_expected は expect(something) によく似ていますが、こちらはワンライナーのテスト(1行で書くテスト)を書くために使われます。<br>

`＊` specify は it のエイリアスです。開発者の中には RSpec の簡潔な構文を使う際に specify を好む人もいます。<br>
     スペックを書いたらそれを声に出して読んでみましょう。そして、一番意味がわかりやすいと思った方を使ってください(訳注:ここで いうわかりやすさとは英文としてのわかりやすさです)。<br>
     この使い分けにはどういう ときにどちらを使うべきかという厳密なルールは存在しません。<br>

is_expected を使うと、次のようなテストが・・・<br>

```rb:sample.rb
it "returns a user's full name as a string" do
  user = FactoryBot.build(:user)
  expect(user.name).to eq "Aaron Sumner"
end
```

以下のように少しだけ簡潔に書き換えることができます。<br>

```rb:sample.rb
subject(:user) { FactoryBot.build(:user) }
it { is_expected.to satisfy { |user| user.name == "Aaron Sumner" } }
```

このようにしても結果は同じです。また、subject はあとに続くテストで暗黙的に再利用 できるので、各 example で何度も打ち直す必要がありません。<br>
とはいえ、私は自分のテストスイートでこの簡潔な構文を使うことはそれほど多くありません。また、subject が一つのテストでしか使われない場合も subject を使いません。<br>
ですが、Shoulda Matchers と一緒に使うのは大好きです。Shoulda というのはそれ自体が単体のテスティングフレームワークで、RSpecと完全に置き換えて使うことができるものでした。<br>
Shoulda はすでに開発が止まっていますが、ActiveModel や ActiveRecord 、ActionController に使える豊富なマッチャのコレクションは独立した gem に切り出され、RSpec (もしくは Minitest )で利用することができます。<br>
この gem を一つ追加すれば、3行から5行あるスペックを1〜2行に減らすことができる場合があります。<br>
Shoulda Matchers を使うには、まず gem を追加しなければなりません。Gemfile 内のテスト 関係の gem のうしろに、この gem を追加してください。<br>

+ `Gemfile`を編集 p164〜<br>

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
  gem 'shoulda-matchers' # 追加 sampleでは既に追加されている
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

コマンドラインから bundle コマンドを実行したら、この新しい gem を使うようにテスト スイートを設定する必要があります。spec/rails_helper.rb を開き、ファイルの一番最後に以下のコードを追加してください。<br>
ここでは、RSpec と Rails で Shoulda Matchers を使うことを宣言しています。<br>

+ `spec/rails_helper.rb`を編集 p165〜<br>

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
  config.include Devise::Test::IntegrationHelpers, type: :system
end

# 追加
Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
# ここまで
```

さあこれでスペックを短くすることができます。User モデルのスペックにある、4つのバリデーションのテストから始めましょう。<br>
Shoulda Matchers を使えば、このテストがたった4行になります!<br>

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

  # ここから編集
  it { is_expected.to validate_presence_of :first_name }
  it { is_expected.to validate_presence_of :last_name }
  it { is_expected.to validate_presence_of :email }
  it { is_expected.to validate_uniqueness_of(:email).case_insensitive }
  # ここまで

  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string" do
    user = FactoryBot.build(:user, first_name: "John", last_name: "Doe")
    expect(user.name).to eq "John Doe"
  end
end
```

+ `$ bundle exec rspec`を実行(パスする)<br>

ここでは Shoulda Matchers が提供する二つのマッチャ(validate_presence_of と validate_- uniqueness_of )を使ってテストを書いています。<br>
email のユニークバリデーションは Devise によって設定されているので、バリデーションは大文字と小文字を区別しないこと(not case sensitive )をスペックに伝える必要があります。<br>
case_insensitive が付いているのはそのためです。<br>
次に Project モデルのスペックに注目してみましょう。<br>
具体的には以前書いた、ユーザーは同じ名前のプロジェクトを複数持つことができないが、ユーザーが異なれば同じ名前のプロジェクトがあっても構わない、というテストです。<br>
Shoulda Matchers を使えば、このテストを1個かつ1行のテストにまとめることができます。<br>

+ `spec/models/project_spec.rb`を編集 @165〜<br>

```rb:project_spec.rb
require 'rails_helper'

RSpec.describe Project, type: :model do
  it { is_expected.to validate_uniqueness_of(:name).scoped_to(:user_id) } # 編集

  # 遅延ステータス
  describe "late status" do
    # 締め切り日が過ぎていれば遅延していること
    it "is late when the due date is past today" do
      project = FactoryBot.create(:project, :due_yesterday)
      expect(project).to be_late
    end

    # 締め切り日が今日ならスケジュールどおりであること
    it "is on time when the due date is today" do
      project = FactoryBot.create(:project, :due_today)
      expect(project).to_not be_late
    end

    # 締め切り日が未来ならスケジュールどおりであること
    it "is on time when the due date is in the future" do
      project = FactoryBot.create(:project, :due_tomorrow)
      expect(project).to_not be_late
    end
  end

  it "can have many notes" do
    project = FactoryBot.create(:project, :with_notes)
    expect(project.notes.length).to eq 5
  end
end
```

私はモデル層でこのようなスタイルのテストを書くのが大好きです。特にモデルをテストファーストで開発するときによく使います。<br>
たとえば、ウィジェット(widget) という新しいモデルが出てきたとします。<br>
そんなとき、私はスペックを開き、ウィジェットがどんな振る舞いを持つのか考えます。それから次のようなテストを書きます。<br>

```rb:sample.rb
it { is_expected.to validate_presence_of :name }
it { is_expected.to have_many :dials }
it { is_expected.to belong_to :compartment }
it { is_expected.to validate_uniqueness_of :serial_number }
```

テストを書いたら、コードを書いてテストをパスさせます。<br>
このアプローチは、どんなコードを書くべきかを考え、それからアプリケーションコードが要件を満たしていることを確認するために非常に役立ちます。<br>

## エディタのショートカット p166〜

プログラミングを習得する上で非常に重要なことは、自分が使っているエディタを隅から 隅まで理解することです。<br>
私はコーディングするときは(そして執筆するときも)たいてい [Atom](https://atom.io/) を使っています。<br>
さらに、[スニペットを作る構文](https://flight-manual.atom.io/using-atom/sections/snippets/)も必要最小限は理解したので、テストを書くときによく使うコードもほとんどタイプせずに済ませることができます。<br>
たとえば、desc と入力してタブキーを押すと、私のエディタは describe...end のブロックを作成します。<br>
さらに、カーソルはブロック内に置かれ、適切な場所にコードを追加できるようになっています。<br>
もしあなたも Atom を使っているのであれば、[Everyday Rails RSpec Atom パッケージ](https://atom.io/packages/atom-everydayrails-rspec) をインストールして私が使っているショートカットを試してみることができます。<br>
全スニペットの内容を知りたい場合はパッケージのドキュメントを参照してください。Atom 以外のエディタを使っている場合は、多少時間をかけてでも用意されているショートカットを使えるようになってください。<br>
また、最初から用意されているショートカットだけでなく、自分自身でショートカットを定義する方法も学習しておきましょう。<br>
お決まりのコードをタイプする時間が少なくなればなるほど、時間あたりのビジネスバリューが増えていきます!<br>

## モックとスタブ

モックとスタブの使用、そしてその背後にある概念は、それだけで大量の章が必要になりそうなテーマです(一冊の本になってもおかしくありません)。<br>インターネットで検索すると、正しい使い方や間違った使い方について人々が激しく議論する場面に出くわすはずです。<br>
また、多くの人々が二つの用語を定義しようとしているのもわかると思います。ただし、うまく定義できているかどうかは場合によりけりです。<br>
私が一番気に入っている定義は次の通りです。<br>

+ __モック(mock)__は本物のオブジェクトのふりをするオブジェクトで、テストのために使われます。<br>
  モックは テストダブル(test doubles) と呼ばれる場合もあります。<br>
  モックはこれまでファクトリや PORO を使って作成したオブジェクトの代役を務めます(訳注: 英語の “double” には「代役」や「影武者」の意味があります)。<br>
  しかし、モックはデータベースにアクセスしない点が異なります。よって、テストにかかる時間は短くなります。<br>

+ スタブ(stub)はオブジェクトのメソッドをオーバーライドし、事前に決められた値を返します。<br>
  つまりスタブは、呼び出されるとテスト用に本物の結果を返す、ダミーメソッドです。<br>
  スタブをよく使うのはメソッドのデフォルト機能をオーバーライドするケースです。特にデータベースやネットワークを使う処理が対象になります。<br>

RSpec には多機能なモックライブラリが最初から用意されています。みなさんはもしかすると、Mocha のようなその他のモックライブラリをプロジェクトで使ったことがあるかもしれません。<br>
第4章以降でテストデータを作成するのに使ってきた Factory Bot にもスタブオブジェクトを作るメソッドが用意されています。<br>
この項では RSpec 標準のモックライブラリに 焦点を当てます。<br>
例をいくつか見てみましょう。メモ(Note)モデルでは delegate を使ってメモに user_name という属性を追加しました。<br>
ここまでに学んだ知識を使うと、この属性がちゃんと機能していることをテストするために、次のようなコードを書くことができます。<br>

+ `spec/modeles/note_spec.rb`を編集 p167〜<br>

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

  # 追加
  # 名前の取得をメモを作成したユーザーに委譲すること
  it "delegates name to the user who created it" do
    user = instance_double("user", name: "Fake User")
    note = Note.new
    allow(note).to receive(:user).and_return(user)
    expect(note.user_name).to eq "Fake User"
  end
  # ここまで
end
```

+ `$ bundle exec rspec`を実行(パスする)<br>

このコードでは User オブジェクトを永続化する必要があります。これはテストで使う first_name と last_name というユーザーの属性にアクセスするためです。<br>
この処理に必要な時間はほんのわずかです。ですが、セットアップのシナリオが複雑になったりすると、わずかな時間もどんどん積み重なって無視できなくなるかもしれません。<br>
モックはこのように、データベースにアクセスする処理を減らすためによく使われます。<br>
さらに、このテストは Noteモデルのテストですが、Userモデルの実装を知りすぎています。<br>
はたしてこのテストの中で Userモデルの name が first_name と last_name から導出されていることを意識する必要があるのでしょうか?<br>
本来であれば関連を持つ User モデルが __name__ という文字列を返すことを知っていればいいだけのはずです。<br>
以下はこのテストの修正バージョンです。このコードでは モックの ユーザーオブジェクトと、テスト対象のメモに設定したスタブメソッドを使っています。<br>

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

  # 名前の取得をメモを作成したユーザーに委譲すること
  it "delegates name to the user who created it" do
    user = double("user", name: "Fake User") # 編集
    note = Note.new
    allow(note).to receive(:user).and_return(user)
    expect(note.user_name).to eq "Fake User"
  end
end
```

ここでは永続化したユーザーオブジェクトをテストダブルに置き換えています。<br>
テストダブルは本物のユーザーではありません。実際、このオブジェクトを調べてみると、Double という名前のクラスになっていることに気づくはずです。<br>
テストダブルは name というリクエストに反応する方法しか知りません。説明のためにエクスペクテーションをテストに一つ追加してみてください。<br>

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

  # 名前の取得をメモを作成したユーザーに委譲すること
  it "delegates name to the user who created it" do
    user = double("user", name: "Fake User")
    note = Note.new
    allow(note).to receive(:user).and_return(user)
    expect(note.user_name).to eq "Fake User"
    expect(note.user.first_name).to eq "Fake" # 追加(確認後削除)
  end
end
```

+ `$ bundle exec rspec spec/models`を実行(失敗する)<br>

このテストコードは元のコード(つまりモックに置き換える前のコード)であればパスしますが、このコードでは失敗します。<br>

```:terminal
Failures:

  1) Note delegates name to the user who created it
     Failure/Error: expect(note.user.first_name).to eq "Fake"
       #<Double "user"> received unexpected message :first_name with (no args)
     # ./spec/models/note_spec.rb:76:in `block (2 levels) in <top (required)>'

Finished in 0.50777 seconds (files took 2.6 seconds to load)
20 examples, 1 failure

Failed examples:

rspec ./spec/models/note_spec.rb:71 # Note delegates name to the user who created it
```

テストダブルは name に反応する方法しか知りません。なぜなら Noteモデルが動作するために必要なコードはそれだけだからです。<br>
というわけで、先ほど追加した expect は削除してください。<br>
さて、次はスタブについて見てみましょう。スタブは allow を使って作成しました。この行はテストランナーに対して、このテスト内のどこかで note.user を呼び出すことを伝えています。<br>
実際に user.name が呼ばれると、note.user_id の値を使ってデータベース上の該当 するユーザーを検索し、見つかったユーザーを返却する代わりに、user という名前のテスト ダブルを返すだけになります。<br>
結果として私たちは、テスト対象のモデルの外部に存在する実装の詳細から独立し、なおかつ2つのデータベース呼び出しを取り除いたテストを手に入れることができました。<br>
このテストはユーザーを永続化することもありませんし、データベース上のユーザーを検索しにいくこともありません。<br>
このアプローチに対する非常に一般的で正しい批判は、もし User#name というメソッドの名前を変えたり、このメソッドを削除したりしても、このテストはパスし続けてしまう、という点です。<br>
みなさんも実際に試してみてください。User#nameメソッドをコメントアウトし、テストを実行してみるとどうなるでしょうか?<br>
ベーシックな RSpec のテストダブルは、代役になろうとするオブジェクトにスタブ化しようとするメソッドが存在するかどうかを検証しません。<br>
この問題を防止するには、かわりに検証機能付きのテストダブル(verified double)を使用します。<br>
このテストダブルが User のインスタンスと同じように振る舞うことを検証してみましょう(ここでは User の最初の1文字が大文字になっている点に注目してください。これは検証機能付きのテストダブルが動作するために必要な変更点です)。<br>

+ `spec/models/note_spec.rb`を編集 p170〜<br>

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

  # 戻す
  # 名前の取得をメモを作成したユーザーに委譲すること
  it "delegates name to the user who created it" do
    user = instance_double("User", name: "Fake User")
    note = Note.new
    allow(note).to receive(:user).and_return(user)
    expect(note.user_name).to eq "Fake User"
  end
end
```

+ 上記は成功する<br>

+ 上記の状態でnameメソッドに何か変更を加えると、テストは失敗します。<br>

```:terminal
1) Note delegates name to the user who created it
Failure/Error: user = instance_double("User", name: "Fake User")
the User class does not implement the instance method: name. Perhaps you meant to use `class_double` instead?
```

今回の実験では、別に class_double を使おうとしていたわけではありません(訳注:エラーメッセージの最後に「もしかすると class_double を使おうとしましたか?」という文言が表示されています)。<br>
ですので、コメントアウトした name メソッドを元に戻して、テストを元通りにすればOKです。<br>

`＊` 失敗メッセージに書かれているとおり、class_double を使うとテストダブルのクラスメソッドも検証することができます。<br>

テスト内でオブジェクトをモック化するためにテストダブルを使う場合、できるかぎり検証機能付きのテストダブルを使うようにしてください。<br>
これを使えば、誤ってテストがパスしてしまう問題を回避することができます。<br>
RSpec で構築された既存のテストスイートが存在するコードベースで開発したことがある人や、他のテストチュートリアルをやったことがある人は、コントローラのテストでモックやスタブが頻繁に使われていることに気づいたかもしれません。<br>
実際、私はコントローラのテストではデータベースにアクセスすることを過剰に避けようとする開発者を過去に何人か見てきました(正直に白状すると、私もやったことがあります)。<br>
以下はデータベースにまったくアクセスせずにコントローラのメソッドをテストする例です。<br>
ここで使われている Noteコントローラの indexアクションはジェネレータが作った元の indexアクションから変更されています。<br>
具体的には、アクションの内部で Noteモデルの searchスコープを呼びだして結果を集め、それをブラウザに返却しています。<br>
この機能をまったくデータベースにアクセスしない形でテストしてみましょう。それがこちらです。<br>

+ `$ bin/rails g rspec:controller notes --controller-specs --no-request-specs`を実行<br>

+ `spec/controllers/notes_controller_spec.rb`を編集 p171〜<br>

```rb:notes_controller_spec.rb
require 'rails_helper'

RSpec.describe NotesController, type: :controller do
  let(:user) { double("user") }
  let(:project) { instance_double("Project", owner: user, id: "123") }

  before do
    allow(request.env["warden"]).to receive(:authenticate!).and_return(user)
    allow(controller).to receive(:current_user).and_return(user)
    allow(Project).to receive(:find).with("123").and_return(project)
  end

  describe "#index" do
    # 入力されたキーワードでメモと検索すること
    it "searches notes by the provided keyword" do
      expect(project).to receive_message_chain(:notes, :search).with("rotate tires")
      get :index,
      params: { project_id: project.id, term: "rotate tires" }
    end
  end
end
```

+ `$ bundle exec rspec spec/controllers`を実行(パスする)<br>

順を追ってコードを見ていきましょう。まず、letを利用してテストで使うuserとproject を遅延定義しています(let については第8章 で説明しました)。<br>
モック化された user に対し てはメソッドをまったく呼び出さないので、ここでは問題なく通常のテストダブルを使うことができます。<br>
一方、project に関しては owner と id の属性を使うので、検証機能付きのテ ストダブルを使った方が安全です。<br>
次に、before ブロックの中では最初に Devise が用意してくれる authenticate! と current_- user メソッドをスタブ化しています。<br>
なぜなら、これはパスワードによって保護されたページだからです。<br>
さらに、Active Record が提供している Project.find メソッドもスタブ化しています。<br>
モック化するのは、データベースを検索するかわりにモックの project を返すためです。<br>
これにより、Project.find(123) がテスト対象のコード内のどこかで呼ばれても、本物のプロジェクトオブジェクトではなく、テストダブルの project が代わりに返されるようになります。<br>

最後に、コントローラのコードが期待どおりに動作することを検証しなければなりません。 <br>
このケースでは、project に関連する notes が持つ search スコープが呼ばれることと、その際の検索キーワード(term )が同名のパラメータの値に一致することを検証しています。<br>
この検証は以下のコードで実現しています。<br>

+ `spec/controllers/notes_controller_spec.rb`(抜粋再掲)<br>

```rb:notes_controller_spec.rb
expect(project).to receive_message_chain(:notes, :search).with("rotate tires")
```

ここでは receive_message_chain を使って project.notes.search を参照しています。<br>
ここで覚えておいてほしいことが一つあります。それは、このエクスペクテーションはアプリケーションコードを実際に動かす前に追加しないとテストがパスしないということです。<br>
ここでは allow ではなく expect を使っているので、もしこのメッセージチェーン(連続したメ ソッド呼び出し)が呼び出されなかった場合はテストが失敗します。<br>
このテストコードや、テスト対象のコントローラのコードでちょっと遊んでみてください。<br>
テストを失敗させる方法を見つけて、RSpec がどんなメッセージを出力するのか調べてみましょう。<br>
さて、ここでこの新しいテストコードの実用性について考えてみましょう。このテストは 速いでしょうか? ええ、ずっと速いはずです。<br>
ユーザーとプロジェクトをデータベース内に作成し、ユーザーをログインさせ、必要なパラメータを使ってコントローラのアクションを呼び出すことに比べれば、このテストは間違いなく速いでしょう。<br>
テストスイートにこのようなテストがたくさんあるなら、データベース呼び出しを必要最小限にすることで、テストの実行がだんだん遅くなる問題を回避することができます。<br>
一方、このコードはトリッキーでちょっと読みづらいです。Devise の認証処理をスタブ化する部分など、セットアップ処理のいくつかは定型的なコードなので、メソッドとして抽出できるかもしれません。<br>
ですが、このセットアップコードはアプリケーションコード内でオブジェクトを作成して保存する方法とはかなり異なるものです。<br>
こうなると、テストコードを読み解くのが大変です。特に初心者は間違いなく苦労することでしょう。<br>
また、ここではモック化に関する有名な原則も無視しています。それは「自分で管理して いないコードをモック化するな([Don’t mock what you don’t own45](https://8thlight.com/insights/thats-not-yours))」という原則です。<br>
つまり、ここでは私たちが自分で管理していないサードパーティライブラリの機能を二つモック化しています。<br>
具体的には Devise が提供している認証レイヤーと、Active Record の find メソッドです。この二つの機能をスピードの向上とコードの分離を目的としてモック化しています。<br>
私たちはこのサードパーティライブラリを自分たちで管理していないにも関わらずモック化しました。<br>
そのため、アプリケーションを壊してしまうような変更がライブラリに入っても、テストコードはその問題を報告してくれないかもしれません。<br>
とはいえ、自分のアプリケーションで管理していないコードをモック化することに意味が あるときもあります。<br>
たとえば、時間のかかるネットワーク呼び出しを実行する必要があったり、レートリミットを持つ外部 API とやりとりする必要があったりする場合、モック化はそうしたコストを最小化してくれます。<br>
テストと Rails の経験が増えてくると、そういったインターフェースと直接やりとりするコードを作成し、アプリケーション内でそのコードが使われている部分をスタブ化するテクニックを有益だと考えるようになるかもしれません。<br>
ですがその場合でも、自分が書いたコードを高コストなインターフェースと直接やりとりさせるテストもある程度残すようにしてください。<br>
この話題は本書の範疇(もしくは本書の小さなサンプルアプリケーションのスコープ)を超えてしまいますが...。<br>
いろいろ話してきましたが、もしあなたがモックやスタブにあまり手を出したくないので あれば、それはそれで心配しないでください。<br>
本書でやっているように、基本的なテストには Ruby オブジェクトを使い、複雑なセットアップにはファクトリを使う、というやり方でも大丈夫です。<br>
スタブはトラブルの原因になることもあります。重要な機能を気軽にスタブ化してしまうと、結果として実際には何もテストしていないことになってしまいます。<br>
テストがとても遅くなったり、再現の難しいデータ(たとえば次の章である程度実践的な例を使って説明する外部 API や web サービスなど)をテストしたりするのでなければ、オブ ジェクトやファクトリを普通に使うだけで十分かもしれません。<br>

## タグ

たとえば、これからこのサンプルアプリケーションに新しい機能を追加することになったとします。<br>
このときに書くテストコードにはいくつかのモデルとコントローラに対する単体テストと、ひとつの統合テストが含まれます。<br>
新しい機能を開発している最中はテストスイート全体を実行するのは避けたいと考えるでしょう。<br>
ですが、各テストを一つずつ実行していくのも面倒です。このような場合は、RSpec のタグ[機能](https://relishapp.com/rspec/rspec-core/docs/command-line/tag-option)が使えます。<br>
タグを使うと、特定のテストだけを実行し、それ以外はスキップするようにフラグを立てることができます。<br>
こうした用途では focus という名前のタグがよく使われます。実行したいテストに対して 次のようにしてタグを付けてください。<br>

```rb:sample.rb
it "processes a credit card", focus: true do
  # example の詳細
end
```

こうすると、コマンドラインから focus タグを持つスペックだけを実行することができます。<br>

+ `$ bundle exec rspec --tag focus`<br>

特定のタグをスキップすることもできます。<br>
たとえば、特別遅い統合テストを今だけ実行したくない場合は、それがわかるようなタグを遅いテストに付けて、それからチルダを付けてタグの指定をひっくり返します(訳注:シェルによっては --tag "∼slow" のようにチル ダ付きのタグをダブルクオートで囲む必要があります)。<br>

+ `$ bundle exec rspec --tag ~slow`<br>

こうすると slow というタグが付いたテスト以外の全テストを実行します。<br>
タグは describe や context ブロックに付けることもできます。その場合はブロック内の全テストにそのタグが適用されます。<br>

`＊`<br>
focus タグはコミットする前に忘れずに削除してください。また、新しい機能が完成 したと判断する前に、必ずタグなしでテストスイート全体を実行するようにしてください。<br>
一方で、slow のようなタグはしばらくコードベースに残しておいても便利かもしれません。<br>

もし、focus タグを頻繁に利用するようになったら、一つでも focus タグの付いたテストが あるときに、そのタグを利用するよう RSpec を設定することもできます。<br>
もし focus タグの 付いた example が一つも見つからなければ、テストスイート全体を実行します。<br>

+ `spec/spec_helper.rb`を確認(p175〜)ただ見るだけ編集しない<br>

```rb:spec_helper.rb
RSpec.configure do |config|
  config.filter_run focus: true
  config.run_all_when_everything_filtered = true
  # 他の設定が続く ...
end
```

特定のタグが付いた example は常にスキップするよう、RSpec を設定することもできます。<br>
たとえば次のように設定します。<br>

+ `spec/spec_helper.rb`を確認(p175〜)ただ見るだけ編集しない<br>

```rb:spec_helper.rb
RSpec.configure do |config|
  config.filter_run_excluding slow: true
  # 他の設定が続く ...
end
```

この場合でもコマンドラインから slow タグの付いたテストだけを明示的に実行することは可能です。<br>

+ `$ bundle exec rspec --tag slow`<br>

## 不要なテストを削除する

もし、あるテストが目的を果たし、今後の回帰テストでも使う必要がないと確信できるなら、そのテストを削除してください!<br>
もし、何かしらの理由でそれを本当に残したいと考えているが、日常的には実行する必要はないなら、そのスペックに skip のマークを付けてください。<br>

```rb:sample.rb
# 大量のデータを読み込むこと
it "loads a lot of data" do
  # 今後不要 (なのでスキップする)
  skip "no longer necessary"

  # スペックのコードが続く。ただし実行はされない
end
```

私はテストをコメントアウトするよりも、スキップする方を推奨します。<br>
なぜなら、スキップされたスペックはテストスイートを実行したときにそのことが表示されるため、スキップしていることを忘れにくくなるからです。<br>
とはいえ、不必要なコードは単純に削除するのが一番良いのは間違いありません(ただし、その確信がある場合に限ります)。<br>
RSpec 3.0より前の RSpec では、この機能は pending メソッドとして提供されていました。<br>
pending は今でも使えますが、機能は全く異なっています。現在の仕様では pending された spec はそのまま実行されます。<br>
そのテストが途中で失敗すると、pending(保留中)として表示されます。しかし、テストがパスすると、そのテストは失敗とみなされます。<br>

## テストを並列に実行する(P176)

私は実行に30分かかるテストスイートを見たことがあります。また、もっと遅いテストス イートがある話もたくさん耳にしました。<br>
すでに遅くなってしまったテストスイートを実行 するよい方法は、[ParallelTests gem](https://github.com/grosser/parallel_tests)を使ってテストを並列に実行することです。<br>
ParallelTests を使うと、テストスイートが複数のコアに分割され、格段にスピードアップする可能性があります。<br>
30分かかるテストスイートを6コアに分けて実行すれば、7分ぐらいまで実行時間が短くなります。これは劇的な効果があります。<br>
特に、遅いテストを個別に改善していく時間がない場合に最適です。<br>

私はこのアプローチが好きですが、注意も必要です。なぜならこのテクニックを使うと、怪しいテストの習慣を隠してしまうことがあるからです。<br>
もしかすると、遅くて高コストな統合テストを使いすぎたりしているのかもしれません。<br>
テストスイートを高速化するときは ParallelTests だけを頼るのではなく、本書で紹介したその他のテクニックも併用するようにしましょう。<br>

## Railsを取り外す

モックやタグの活用はどちらもテストスイートの実行時間を減らすためのものでした。しかし究極的には、処理を遅くしている大きな要因の一つは Rails自身です。<br>
テストを実行するときは毎回 Rails の一部、もしくは全部を起動させる必要があります。<br>
Spring gem をインストールし(ただし、Rails 6.1以前であればデフォルトでインストールされています)、binstub を使って Spring 経由で RSpec を実行すれば、起動時間を短くすることができます。<br>
しかし、 もし本当にテストスイートを高速化したいなら、Rails を完全に取り外すこともできます。 <br>
Spring を使うとフレームワークを読み込んでしまいますが(ただし一回だけ)、フレームワークを読み込まないこちらのソリューションなら、フレームワークをまったく読み込まないので、さらにもう一歩高速化させることができます。<br>
この話題は本書の範疇を大きく超えてしまいます。なぜならあなたのアプリケーションアーキテクチャ全体を厳しく見直す必要があるからです。<br>
また、これは Rails 初心者に説明する際の個人的なルールも破ることになります。<br>
つまり、Rails の規約を破ることは可能な限り避けなければならない、という私自身のルールも破ることになるわけです。<br>
それでもこの内容について詳しく知りたいのであれば、Corey Haines の講演と[Destroy All Software](https://www.destroyallsoftware.com/screencasts)を視聴する ことをお勧めします。<br>

## まとめ

この章にやってくる前は、テストのやり方に複数の選択肢があることをほとんど説明してきませんでした。<br>
しかし、これであなたは複数の選択肢を選べるようになりました。あなたはあなた自身とあなたのチームにとって一番良い方法を選び、スペックを明快なドキュメントにすることができます。<br>
第3章で説明したような冗⻑な書き方を使うこともできますし、ここで説明したようなもっと簡潔な方法を使うこともできます。<br>
あなたはモックとスタブ、ファクトリ、ベーシックな Ruby オブジェクト、それらの組み合わせ等々、様々な方法を選べるはずです。<br>
最後にあなたはテストスイートをロードし、それを実行するオプションも覚えました。<br>
さあ、ゴールは目前です!あといくつかのカバーすべき話題を見たら、テストプロセスの全体像と落とし穴の避け方(これには初めて登場するテストファースト形式の開発プロセスも含まれます)を説明して本書を締めくくります。<br>
ですが、その前に典型的な web アプリケーションでよくある、私たちがまだテストしていないマイナーな機能について見ていきましょう。<br>

## 演習問題

+ あなたのテストスイートの中で、この章で説明した簡潔な構文を使ってきれいにできるスペックを探してください。<br>
  この演習問題をやってみることで、あなたのスペックはどれくらい短くなるでしょうか?<br>
  読みやすさも同じでしょうか?あなたは簡潔な書き方と冗⻑な書き方のどちらが好みでしょうか?(ヒント: 最後の問いには正解はありませ ん。)<br>

+ shoulda-matchers をインストールし、それを使ってスペックをきれいにできそうな場所を探してください(またはテストしていない部分をテストしてください)。<br>
  [GitHub にある gem のソースコード](https://github.com/thoughtbot/shoulda-matchers)をチェックし、これらのマッチャがどのように実装されているのか詳しく学習してください。<br>

+ RSpec のタグを使って、遅いスペックに目印を付けてください。<br>
  そのテストを含めたり 除いたりしてテストスイートを実行してください。どれくらいパフォーマンスが 向上するでしょうか?<br>
