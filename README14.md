# 10 その他のテスト p178〜

私たちはこれまで、プロジェクト管理アプリケーション全体に対して堅牢なテストスイートを構築してきました。<br>
モデルとコントローラをテストし、システムスペックを通じて view と組み合わせた場合もテストしました。<br>
リクエストスペックを使った、外部向け API のテストもあります。しかし、まだカバーしていないテストが少し残っています。<br>
メモ(Note)モ デルにあるファイルアップロード機能や、メール送信機能、ジオコーディング(緯度経度) 連携のテストはどうすればいいでしょうか?こうした機能もテストできます!<br>
この章では次のような内容を説明します。<br>

+ ファイルアップロードのテスト<br>

+ Active Job を使ったバックグラウンドジョブのテスト<br>

+ メール送信のテスト方法<br>

+ 外部 Web サービスに対するテスト<br>

## ファイルアップロードのテスト

添付ファイルのアップロードは Web アプリケーションでよくある機能要件の一つです。Rails 5.2からは Active Storage と呼ばれるファイルアップロード機能が標準で提供されるようになりました。<br>
本書のサンプルアプリケーションでも Active Storage を使用しています。このほかにも CarrierWave や Shrine といった gem を使ってファイルアップロード機能を実装する方法もあります(もしくは自分で独自に作るという手もあります)。<br>
しかし、どうやってテストすれば良いでしょうか? 本書では Active Storage を使ったテスト方法を紹介しますが、基本的なアプローチは他のライブラリを使う場合でも同じです。<br>
では、メモ機能をテストするシステムスペックから始めましょう。<br>

+ `$ rails g rspec:system notes`を実行<br>

+ `spec/system/notes_spec.rb`を編集<br>

```rb:notes_spec.rb
require 'rails_helper'

RSpec.describe "Notes", type: :system do
  let(:user) { FactoryBot.create(:user) }
  let(:project) {
    FactoryBot.create(:project,
      name: "RSpec tutorial",
      owner: user)
  }

  # ユーザーが添付ファイルをアップロードする
  scenario "user uploads an attachment" do
    sign_in user
    visit project_path(project)
    click_link "Add Note"
    fill_in "Message", with: "My book cover"
    attach_file "Attachment", "#{Rails.root}/spec/files/attachment.jpg"
    click_button "Create Note"
    expect(page).to have_content "Note was successfully created"
    expect(page).to have_content "My book cover"
    expect(page).to have_content "attachment.jpg(imge/jpeg"
  end
end
```

これは今まで書いてきた他のシステムスペックととてもよく似ています。ですが、今回は Capybara の attach_file メソッドを使って、ファイルを添付する処理をシミュレートしています。<br>
最初の引数は入力項目のラベルで、二つ目の引数がテストファイルのパスです。ここでは spec/files という新しいディレクトリが登場しています。<br>
この中にテストで使う小さな JPEG ファイルが格納されています。このディレクトリ名は何でも自由に付けられます。<br>
他にはたとえば、spec/fixtures という名前が付けられることもあるようです。ファイルの名前も自由です。<br>
ただし、このファイルは忘れずにバージョン管理システムにコミットしておいてください。そうしないと他の開発者がテストを実行したときに、テストが失敗してしまうからです。<br>
テストファイルが指定された場所に存在すれば、この新しいスペックは問題なくパスするはずです。<br>
ところで、Active Storage では config/storage.yml でアップロードしたファイルの保存先を指定します。<br>
特に変更していなければ、テスト環境では tmp/storage になっているはずです。<br>

+ `config/storage.yml`を参照<br>

```config/storage.yml
test:
  service: Disk
  root: <%= Rails.root.join("tmp/storage") %>

local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

# Use bin/rails credentials:edit to set the AWS secrets (as aws:access_key_id|secret_access_key)
# amazon:
#   service: S3
#   access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
#   secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
#   region: us-east-1
#   bucket: your_own_bucket-<%= Rails.env %>

# Remember not to checkin your GCS keyfile to a repository
# google:
#   service: GCS
#   project: your_project
#   credentials: <%= Rails.root.join("path/to/gcs.keyfile") %>
#   bucket: your_own_bucket-<%= Rails.env %>

# Use bin/rails credentials:edit to set the Azure Storage secret (as azure_storage:storage_access_key)
# microsoft:
#   service: AzureStorage
#   storage_account_name: your_account_name
#   storage_access_key: <%= Rails.application.credentials.dig(:azure_storage, :storage_access_key) %>
#   container: your_container_name-<%= Rails.env %>

# mirror:
#   service: Mirror
#   primary: local
#   mirrors: [ amazon, google, microsoft ]
```

tmp/storage ディレクトリを開いて、なんらかのディレクトリやファイルが作成されていることを確認してみてください。<br>
ただし、ディレトリ名やファイルは推測されにくい名前になっているため、どのファイルがどのテストで作成されたのか探し当てるのは少し難しいかもしれません。<br>
今のままだと、テストを実行するたびにファイルが増えていってしまうため、テストの実行が終わったら、アップロードされた古いファイルは自動的に削除されるようにしておくと良いと思います。<br>
spec/rails_helper.rb に以下の設定を追加してください。<br>

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
  config.include Devise::Test::IntegrationHelpers, type: :system

  # 追加
  # テストスイートの実行が終わったらアップロードされたファイルを削除する
  config.after(:suite) do
    FileUtils.rn_rf("#{Rails.root}/tmp/storage")
  end
  # ここまで
end


Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
```

+ `$ mkdir spec/files && touch $_/attachment.jpg`を実行<br>

さあ、これでテストが終わった時点で RSpec がこのディレクトリとその中身を削除してくれます。<br>
さらに、偶然この中のファイルがバージョン管理システムにコミットされてしまわないよう、プロジェクトの .gitignore に設定を追加しておくことも重要です。<br>
Active Storage の場合は Rails が最初から設定を追加してくれているので、念のため確認するだけで OK ですが、他のライブラリを使ってアップロード機能を実現するときはこうした設定追加を忘れないようにしてください。<br>

+ `$ bundle exec rspec spec/system`を実行(パスする)<br>

+ `.gitignore`を編集<br>

```:.gitignore
# Ignore uploaded files in development. /storage/*
!/storage/.keep
/tmp/storage/*
!/tmp/storage/ !/tmp/storage/.keep
```

さて、Active Storage のことはいったん忘れて、ここでやった内容を振り返ってみましょう。<br>
最初はファイルアップロードのステップを含む、スペックファイルを作成しました。さらに、スペック内ではテストで使うファイルを添付しました。<br>
次に、テスト専用のアップロードパスを設定(確認)しました。最後に、テストスイートが終わったらファイルを削除するように RSpec を設定しました。<br>
この3つのステップはシステムレベルのテストでファイルアップロードを実行するための基本的なステップです。<br>
もしみなさんが Active Storage を使っていない場合は、自分が選んだアップロード用ライブラリのドキュメントを読み、この3つのステップを自分のアプリケーションに適用する方法を確認してください。<br>
もし自分でファイルアップロードを処理するコードを書いている場合は、アップロード先のディレクトリを変更するために、allow メソッド(第9章を参照)でスタブ化できるかもしれません。<br>
もしくはアップロード先のパスを実行環境ごとの設定値として抜き出す方法もあるでしょう。<br>
最後に、もしテスト内で何度もファイル添付を繰り返しているのであれば、そのモデルをテストで使う際に、添付ファイルを属性値に含めてしまうことを検討した方がいいかもしれません。<br>
たとえば、Note ファクトリにこのアプローチを適用する場合は、第4章で説明したトレイトを使って実現することができます。<br>

+ `spec/factories/notes.rb`を編集<br>

```rb:notes.rb
FactoryBot.define do
  factory :note do
    message { "My important note." }
    association :project
    user { project.owner }

    trait :with_attachment do
      attachment { Rack::Test::UploadedFile.new("#{Rails.root}/spec/files/attachment.jpg", 'image/jpeg') }
    end
  end
end
```

Active Storage を使っている場合は上のように Rack::Test::UploadedFile.new の引数に 添付ファイルが存在するパスと Content-Type を指定します。<br>
こうすればテスト内で FactoryBot.create(:note, :with_attachment) のように書くことで、ファイルが最初から添付された新しい Note オブジェクトを作成することができます。<br>
モデルスペックを開き、次の テストを追加してください。<br>

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

  # 添付ファイルを1件添付できる
  it 'has one attached attachment' do
    note = FactoryBot.create(:note, :with_attachment)
    expect(note.attachment).to be_attached
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

    # 追加
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
    # ここまで
  end

  # 名前の取得をメモを作成したユーザーに委譲すること
  it "delegates name to the user who created it" do
    user = instance_double("user", name: "Fake User")
    note = Note.new
    allow(note).to receive(:user).and_return(user)
    expect(note.user_name).to eq "Fake User"
  end
end
```

+ `$ bundle exec rspec`を実行(パスする)<br>

`＊`<br>

このアプローチはよく考えて使ってください!このファクトリは使うたびにファイルシステムへの書き込みが発生するので、遅いテストの原因になります。<br>

## バックグラウンドワーカーのテスト p183〜<br>

私たちのプロジェクト管理アプリケーションを利用している、架空の会社の架空のマーケティング部が、CM を流したり、広告を打ったりする参考情報を得るために、私たちに対してユーザーに関する位置情報を集めてくるように依頼してきたとします。<br>
個人情報の扱いや利用規約に関する詳細は法務部に任せるとして、とりあえず私たちは location 属性をユーザーモデルに追加してこの機能を完成させました。<br>
この機能はユーザーがログインしたときに外部のジオコーディングサービスにアクセスし、町や州、国といった情報を取得します。<br>
今回新たに加わったこの処理は Active Job を使ってバックグラウンドで実行されます。<br>
そのため、ユーザーはこの処理が終わるまで待たされることはありません。<br>

`＊`<br>
この機能はサンプルアプリケーションに実装済みですが、Rails の開発環境では少しトリッキーな実装になっています。<br>
なぜなら、開発環境ではログインしたときに localhost や 127.0.0.1 になってしまうからです。<br>
私は偽物のランダムな IP アドレスを持ったサンプルデータ(シードデータ)を追加しています。<br>
これは Rails コンソールで試すことが可能です。<br>
まず、bin/rails db:seed を実行し、シードデータを開発環境に追加してください。<br>
それから bin/rails console で Rails コンソールを開き、ユーザーを一人選んでください。<br>
ユーザーを選んだら、geocode メソッドを呼んでください。<br>

```:terminal
> u = User.find(10)
> u.location
=> nil
> u.geocode
=> true
> u.location
=> "Johannesburg, Gauteng, South Africa"
```

+ `$ rails g rspec:system sign_ins`を実行<br>

+ `spec/system/sign_ins_spec.rb`を編集 p185〜<br>

```rb:sign_ins_spec.rb
require 'rails_helper'

RSpec.describe "Sign in", type: :system do
  let(:user) { FactoryBot.create(:user) }

  before do
    ActiveJob::Base.queue_adapter = :test
  end

  # ユーザーのログイン
  scenario "user signs in" do
    visit root_path
    click_link "Sign In"
    fill_in "Email", with: user.email
    fill_in "Password", with: user.password
    click_button "Log in"

    expect {
      GeocodeUserJob.perform_later(user)
    }.to have_enqueued_job.with(user)
  end
end
```

このコードには Active Job とバックグラウンドのジオコーディング処理に関連する部分が2箇所あります。<br>
まず、rspec-rails ではバックグラウンドジョブをテストするために、queue_adapter に :test を指定する必要があります。<br>
これがないとテストは次のような例外を理由付きで raise します(訳注:下の例外メッセージには「ActiveJob マッチャを使うには、 ActiveJob::Base.queue_adapter = :test を設定してください」と書いてあります)。<br>

```
StandardError:
To use ActiveJob matchers set `ActiveJob::Base.queue_adapter = :test`
```

ここではよく目立つように before ブロックでこのコードを実行しましたが、scenario の 中で実行しても構いません。<br>
なぜなら、このファイルにはテストが一つしかないからです。<br>
また、複数のファイルでテストキューを使う場合は、第8章 で紹介したテストを DRY にする テクニック(訳注:shared_context のこと)を使って実験することもできます。<br>

次に、ジョブが実際にキューに追加されたことを確認する必要があります。<br>
rspec-rails では この確認に使えるマッチャがいくつか用意されています。<br>
ここでは have_enqueued_job を使い、正しいジョブが正しい入力値で呼ばれていることをチェックしています。<br>
注意してほしいのは、このマッチャはブロックスタイルの expect と組み合わせなければいけないことです。<br>
こうしないと、テストは次のような別の例外を理由付きで raise します(訳注:下の例外 メッセージには「have_enqueued_job と enqueue_job はブロック形式のエクスペクテーションだけをサポートします」と書いてあります)。<br>

```
ArgumentError:
  have_enqueued_job and enqueue_job only support block expectations
```

+ `＊`<br>

have_enqueued_job マッチャはチェイニングする(他のマッチャと連結させる)こと により、キューの優先度やスケジュール時間を指定することもできます。<br>
この[マッチャのオンラインドキュメント](https://www.rubydoc.info/gems/rspec-rails/RSpec/Rails/Matchers:have_enqueued_job) には数多くの実行例が載っています。<br>

さて、これでバックグラウンドジョブがアプリケーションの他の部分と正しく連携できていることをチェックできました。<br>
今度はもっと低レベルのテストを書いて、ジョブがアプリケーション内のコードを適切に呼びだしていることを確認しましょう。<br>
このジョブをテストするために、新しいテストファイルを作成します。<br>

+ `$ bin/rails g rspec:job geocode_user`を実行<br>

この新しいファイルは本書を通じて作成してきた他のテストファイルとよく似ています。<br>

+ `spec/jobs/geocode_user_job_spec.rb`<br>

```rb:geocode_user_jpb_spec.rb
require 'rails_helper'

RSpec.describe GeocodeUserJob, type: :job do
  pending "add some examples to (or delete) #{__FILE__}"
end
```

+ `spec/jobs/gecode_user_job.rb`を編集(P187〜)<br>

```rb:geocode_user_job.rb
require 'rails_helper'

RSpec.describe GeocodeUserJob, type: :job do
  # user の geocodeを呼ぶこと
  it "calls geocode on the user" do
    user = instance_double("User")
    expect(user).to receive(:geocode)
    GeocodeUserJob.perform_now(user)
  end
end
```

+ `$ bundle exec rspec`を実行(パスする)<br>

このテストでは第9章で説明した instance_double を使ってテスト用のモックユーザーを作っています。<br>
それからテスト実行中のどこかのタイミングでこのモックユーザーに対して geocode メソッドが呼び出されることを RSpec に伝えています。<br>
最後に、perform_now メソッドを使って、このバックグラウンドジョブ自身を呼び出します。<br>
こうすると、ジョブはキューに入らないため、テストの実行結果をすぐに検証できます。<br>

+ `＊`<br>

この geocode というインスタンスメソッドは Geocoder gem によって定義されています。<br>
サンプルアプリケーションでは User モデル内で geocoded_by というメソッドを使ってこの gem を利用しています。<br>
このテストでは Geocoder のことを詳しく知らなくても問題ありませんが、練習のためにもっとテストを書いてみたいと思った場合は、この[gem のドキュメント](http://www.rubygeocoder.com/)を読むとよく理解できるはずです。<br>

## メール送信をテストする

フルスタックWebアプリケーションの多くは何らかのメールをユーザーに送信します。<br>
このサンプルアプリケーションでは短いウェルカムメッセージに、アプリケーションのちょっとした使い方を添えて送信します。<br>
メール送信は二つのレベルでテストできます。一つはメッセージがただしく構築されているかどうかで、もう一つは正しい宛先にちゃんと送信されるかどうかです。<br>
このテストでは先ほど新しく学んだ、バックグラウンドワーカーのテストの知識を使います。<br>
なぜならメール送信はバックグラウンドワーカーを利用することが一般的だからです。<br>
最初は Mailer にフォーカスしたテストから始めましょう。<br>
今回対象となる Mailerは app/mailers/user_mailer.rb にある Mailer です。<br>
このレベルのテストでは、送信者と受信者のアドレスが正しいことと、件名とメッセージ本文に大事な文言が含まれていることをテストし ます。(もっと複雑な Mailer になると、今回の基本的な Mailer よりも、もっとたくさんの項 目をテストすることになるはずです。)<br>
新しいテストファイルはジェネレータを使って作成します。<br>

+ `$ bin/rails g rspec:mailer user_mailer`を実行<br>

この新しいファイルで注目したいのは、spec/mailers/user_mailer.rb にファイルが作成されている点と、次のような定型コードが書かれている点です。<br>

+ `spec/mailers/user_mailer_spec.rb`を編集<br>

```rb:user_mailer_spec.rb
require "rails_helper"

RSpec.describe UserMailer, type: :mailer do
  describe "welcome_email" do
    let(:user) { FactoryBot.create(:user) }
    let(:mail) { UserMailer.welcome_email(user) }

    # ウェルカムメールをユーザーのメールアドレスに送信すること
    it "sends a welcome email to the user's email address" do
      expect(mail.to).to eq [user.email]
    end

    # サポート用のメールアドレスから送信すること
    it "sends a welcome email to the user's email address" do
      expect(mail.to).to eq [user.email]
    end

    # サポート用のメールアドレスから送信すること
    it "sends from the support email address" do
      expect(mail.from).to eq ["support@example.com"]
    end

    # 正しい件名で送信すること
    it "sends with the correct subject" do
      expect(mail.subject).to eq "Welcome to Projects!"
    end

    # ユーザーにはファーストネームで挨拶すること
    it "greets the user by first name" do
      expect(mail.body).to match(/Hello #{user.first_name},/)
    end

    # 登録したユーザーのメールアドレスを残しておくこと
    it "reminds the user of the registered email address" do
      expect(mail.body).to match user.email
    end
  end
end
```

+ `$ bundle exec rspec`を実行(パスする)<br>

ここではテストデータ、すなわち、テスト対象のユーザーと Mailer のセットアップから始まっています。<br>
それから実装された仕様を確認するための小さな単体テストをいくつか書いています。<br>
最初に書いたのは mail.to のアドレスを確認するテストです。ここで注意してほしいのは、mail.to の値は文字列の配列になる点です。<br>
単体の文字列ではありません。mail.from も同様にテストします。このテストでは RSpec の contain マッチャを使うこともできますが、私は配列の等値性をチェックする方が好みです。<br>
こうすると余計な受信者や送 信者が含まれていないことを確実に検証できます。<br>
mail.subject のテストはとても単純だと思います。ここまでは eq マッチャを使って何度も文字列を比較してきました。<br>
ですが、mail.body を検証する最後の二つのテストでは match マッチャを使っている点が少し変わっています。<br>
このテストではメッセージ本文全体をテストする必要はなく、本文の一部をテストすればいいだけです。<br>
最初の example では正規表現を使って、フレンドリーなあいさつ(たとえば、Hello, Maggie, のようなあいさつ)が本文に含まれていることを確認しています。<br>
二つ目の example でも match を使っていますが、この場合は正規表現を使わずに、本文のどこかに user.email の文字列が含まれることを確認しているだけです。<br>

+ `＊`<br>

もしメールに関連するスペックをもっと表現力豊かに書きたいのであれば、[Email Spec](https://rubygems.org/gems/email_spec) というライブラリをチェックしてみてください。<br>
このライブラリは deliver_to や have_body_text といったマッチャを提供してくれます。<br>
これを使えば、みなさんのテストがもっと読みやすくなるかもしれません。<br>

繰り返しになりますが、みなさんが書く Mailer のスペックの複雑さは、みなさんが作ったMailer の複雑さ次第です。<br>
今回はこれぐらいで十分かもしれませんが、もっとたくさんテストを書いた方が Mailer に対して自信が持てるのであれば、そうしても構いません。<br>
さて、今度はアプリケーションの大きなコンテキストの中で Mailer をテストする方法を見ていきましょう。<br>
このアプリケーションでは新しいユーザーが作られると、そのたびにウェルカムメールが送信される仕様になっています。<br>
どうすれば、本当に送信されていることを検証できるでしょうか? 高いレベルでテストするなら統合テストでテストできますし、もう少し低いレベルでテストするならモデルレベルでテストできます。<br>
練習として、両方のやり 方を見ていきましょう。最初は統合テストから始めます。<br>
このメッセージはユーザーのサインアップワークフローの一部として送信されます。<br>
それではこのテストを書くための新しいシステムスペックを作成しましょう。<br>

+ `$ bin/rails g rspec:system sign_up`を実行<br>

次に、ユーザーがサインアップするためのステップと期待される結果をこのファイルに追加します。<br>

+ `spec/system/sign_ups_spec.rb`を編集(P190〜191)<br>

```rb:sign_ups_spec.rb
require 'rails_helper'

RSpec.describe "Sign-ups", type: :system do
  include ActiveJob::TestHelper

  # ユーザーはサインアップに成功する
  scenario "user successfully sins up" do
    visit root_path
    click_link "Sign up"

    perform_enqueued_jobs do
      expect {
        fill_in "First name", with: "First"
        fill_in "Last name", with: "Last"
        fill_in "Email", with: "test@example.com"
        fill_in "Password", with: "test123"
        fill_in "Password confirmation", with: "test123"
        click_button "Sign up"
      }.to change(User, :count).by(1)

      expect(page).to have_content "Welcome! You have signed up successfully."
      expect(current_path).to eq root_path
      expect(page).to have_content "First Last"
    end

    mail = ActionMailer::Base.deliveries.last

    aggregate_failures do
      expect(mail.to).to eq ["test@example.com"]
      expect(mail.from).to eq ["support@example.com"]
      expect(mail.subject).to eq "Welcome to Projects!"
      expect(mail.body).to match "Hello First,"
      expect(mail.body).to match "test@example.com"
    end
  end
end
```

+ `$ bundle exec rspec`を実行(パスする)<br>

このテストの約3分の2は第6章 以降でよく見慣れたものかもしれません。ここでは Capybara を使ってサインアップフォームへの入力をシミュレートしています。<br>
残りの部分はメール送信にフォーカスしています。メールはバックグラウンドプロセスで送信されるため、テストコードは perform_enqueued_jobs ブロックで囲む必要があります。<br>
このヘルパーメソッドは、このスペックファイルの最初で include している ActiveJob::TestHelper モジュールが提供しています。<br>

このメソッドを使えば ActionMailer::Base.deliveries にアクセスし、最後の値を取ってくることができます。<br>
この場合、最後の値はユーザーが登録フォームに入力したあとに送信されるウェルカムメールになります。<br>
テストしたいメールオブジェクトを取得できれば、残りのエクスペクテーションは Mailer に追加した単体テストとほとんど同じです。<br>
実際のところ、統合テストのレベルではここまで詳細なテストは必要ないかもしれません。<br>
ここでは mail.to をチェックして適切なユーザーにメールが送信されていることと、mail.subject をチェックして適切なメッセージが送信されていることを検証するだけで十分かもしれません。<br>
なぜならその他の詳細なテストは Mailer のスペックに書いてあるからです。これはただ私が RSpec にはこういうやり方もあるということを紹介したかっただけです。<br>
このテストは User モデルと UserMailer が連携するポイントを直接テストすることでも実現できます。<br>
このアプリケーションでは新しいユーザーが追加されたときに after_create コールバックでこの連携処理が発生するようになっています。なので、ユーザーのモデルスペックにテストを追加することができます。<br>

+ `spec/models/user_spec.rb`を編集(P192〜)<br>

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

  it { is_expected.to validate_presence_of :first_name }
  it { is_expected.to validate_presence_of :last_name }
  it { is_expected.to validate_presence_of :email }
  it { is_expected.to validate_uniqueness_of(:email).case_insensitive }

  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string" do
    user = FactoryBot.build(:user, first_name: "John", last_name: "Doe")
    expect(user.name).to eq "John Doe"
  end

  # 追加
  # アカウントが作成された時にウェルカムメールを送信すること
  it "sends a welcome email an account creation" do
    allow(UserMailer).to receive_message_chain(:welcome_email, :deliver_later)
    user = FactoryBot.create(:user)
    expect(UserMailer).to have_received(:welcome_email).with(user)
  end
  # ここまで
end
```

+ `テストはパスする`<br>

このテストではまず、receive_message_chain を使って deliver_later メソッドをスタブ化しています。<br>
このメソッドは Action Job が UserMailer.welcome_email に提供してくれるメソッドです。receive_message_chain メソッドについては、第9章 でも同じような使い方を説明しています。<br>

次に、テストするユーザーを作成する必要があります。Mailer はユーザーを作成する処理の一部として実行されるため、その直後にスパイ(spy) を使ってテストします。<br>
スパイは 第9章で説明したテストダブルによく似ていますが、テストしたいコードが実行されたあとに発生した何かを確認できる点が異なります。<br>
具体的には、クラス(UserMailer )に対して、期待されたオブジェクト(user)とともに、目的のメッセージ(:welcome_email )が呼び出されたかどうかを、have_received マッチャを使って検証しています。<br>
なぜスパイを使う必要があるのでしょうか? これは次のような問題を回避するためです。<br>
すなわち、ここではユーザーを作成してそれを変数に入れる必要がありますが、そうするとテストしようとしている Mailer も実行されてしまいます。<br>
つまり、次のようなコードを書いても動かないということです。<br>

```rb:sample.rb
it "sends a welcome email on account creation" do
  expect(UserMailer).to receive(:welcome_email).with(user)
  user = FactoryBot.create(:user)
end
```

これは以下のエラーを出して失敗します。<br>

```:termianl
Failures:

  1) User sends a welcome email on account creation
     Failure/Error: expect(UserMailer).to
       receive(:welcome_email).with(user)

  NameError:
    undefined local variable or method `user' for
    #<RSpec::ExampleGroups::User:0x007fca733cb578>
```

user という変数が作成される前にこの変数を使うことはできません。ですが、テスト対象 のコードを実行せずに user を作成することもできません。<br>
幸いなことに、スパイを使えばどちらでもない新たなワークフローを選択することができます。<br>
このテストのいいところは、Mailer が正しい場所で、正しい情報とともに呼ばれることを確認するだけで十分であることです。<br>
このテストではいちいち Web 画面を使ってユーザーを作成する必要がありません。そのため速く実行できます。ですが、短所もあります。<br>
この テストは[レガシーコードをテストするために提供されている RSpec のメソッド](https://relishapp.com/rspec/rspec-mocks/docs/working-with-legacy-code/message-chains)を使っている点です。また、スパイを使うとこのテストコードを初めて見た開発者をびっくりさせてしまうかもしれません。<br>
また、このワークフローを一度見直してみるのもいいかもしれません。たとえば deliver_later を使ってウェルカムメールを送信するのではなく、別のバックグラウンドジョブを使って送信するようにしたり、after_create コールバックを削除したりすることも検討してみましょう。<br>
このようなケースでは高いレベルの統合テストを残した状態で低レベルのテストを作成し、アプリケーションコードが適切に連携できていることを確認できたら、統合テストの方は削除してもいいかもしれません。<br>
繰り返しになりますが、開発者には選択肢がいろいろあるのです。<br>

## Webサービスをテストする(P194〜)

ではジオコーディングに戻りましょう。バックグラウンドジョブのテストは作成しましたが、実装の詳細はまだちゃんとテストできていません。<br>
ジオコーディングの処理は本当に実行されているのでしょうか? 現在の実装コードでは、ユーザーが正常にログインしたあとにジオコーディングがバックグラウンドで実行されます。<br>
ですが、それはユーザーが実行中のアプリケーションサーバーと異なるホストからログインした場合だけです。<br>
位置情報はIPアドレスに基づいてリクエストされます。そして、繰り返しになりますが、ジオコーディングはアプリケーションの中で実行されているのではありません。<br>
この処理はHTTPコールを経由して外部のジオコーディングサービスに実行してもらっているのです。<br>
そこでテストを追加して本当にジオコーディングサービスにアクセスしていることを検証しましょう。既存の User モデルのスペックに新しいテストを追加してください。<br>

+ `spec/models/user_spec.rb`を編集<br>

```rb:sample.rb
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

  it { is_expected.to validate_presence_of :first_name }
  it { is_expected.to validate_presence_of :last_name }
  it { is_expected.to validate_presence_of :email }
  it { is_expected.to validate_uniqueness_of(:email).case_insensitive }

  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string" do
    user = FactoryBot.build(:user, first_name: "John", last_name: "Doe")
    expect(user.name).to eq "John Doe"
  end

  # アカウントが作成された時にウェルカムメールを送信すること
  it "sends a welcome email an account creation" do
    allow(UserMailer).to receive_message_chain(:welcome_email, :deliver_later)
    user = FactoryBot.create(:user)
    expect(UserMailer).to have_received(:welcome_email).with(user)
  end

  # 追加
  # ジオコーディングを実行すること
  it "performs geocoding" do
    user = FactoryBot.create(:user, last_sign_in_ip: "161.185.207.20")
    expect {
      user.geocode
    }.to change(user, :location).
      from(nil).
      to("Brooklyn, New York, US")
  end
  # ここまで
end
```

このテストは第3章で書いた他のテストによく似ています。<br>
静的なIPアドレスを事前に設定したユーザーを作成し、ユーザーに対してジオコーディングを実行し、ユーザーの位置情報が取得できたことを検証しています(ちなみに私は上の実験でこのIPアドレスに対応する位置情報をたまたま知りました)。<br>
スペックを実行すると、このテストはパスします(訳注:ジオコーディングサービスが返す位置情報が変更されるとテストが失敗することがあります。<br>
その場合はテストコードを修正してテストがパスするようにしてください)。ですが、一つ問題があります。この小さなスペックは同じファイルに書かれた他のテストよりも明らかに遅いのです。<br>
なぜだかわかりますか? このテストではジオコーディングサービスに対して実際にHTTPリクエストを投げます。そのため、サービスが位置情報を返すまで待ってから新しい値を確認しなければならないのです。<br>
実行速度が遅くなることに加えて、外部のサービスを直接使ってテストすることは別のコストも発生させます。<br>
もしこの外部APIにレートリミットが設定されていたら、レートリミットを超えたタイミングでテストが失敗し始めます。<br>
また、これが有料のサービスだった場合、テストを実行することで実際に利用料金が発生してしまうかもしれません!<br>
[VCR](https://github.com/vcr/vcr) gem はこういった問題を軽減してくれる素晴らしいツールです。<br>
VCR を使えばテストを高速に保ち、APIの呼び出しを必要最小限に抑えることができます。VCR は Ruby コードから送られてくる外部へのHTTPリクエストを監視します。<br>
そうした HTTPリクエストが必要なテストが実行されると、そのテストは失敗します。テストをパスさせるには、HTTP通信を記録する “カセット” を作る必要があります。<br>
テストをもう一度実行すると、VCR はリクエストとレスポンスをファイルに記録します。こうすると今後、同じリクエストを投げるテストはファイルから読み取ったデータを使うようになります。<br>
外部の API に新たなリクエストを投げることはありません。<br>
では VCR をアプリケーションに追加することから始めましょう。まず、この VCR と VCR によって利用される WebMock を Gemfile に追加してください。<br>

+ `Gemfile`を編集(サンプルは導入済み)<br>

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
  gem 'shoulda-matchers'
  gem 'vcr' # 追加
  gem 'webmock' # 追加
end

gem 'devise'
gem 'net-imap'
gem 'net-pop'
gem 'net-smtp'
gem 'activestorage-validator'
gem 'geocoder'
```

[WebMock](https://github.com/bblimke/webmock)は HTTP をスタブ化するライブラリで、VCR は処理を実行するたびにこのライ ブラリを水面下で利用しています。WebMock はそれ自体がパワフルなツールですが、説明をシンプルにするため、ここでは詳しく説明しません。<br>
bundle install を実行して、新しい gem を追加してください。<br>
次に、外部に HTTP リクエストを送信するテストで VCR を使うように RSpec を設定する必要があります。<br>
spec/support/vcr.rb というファイルを新たに作成し、次のようなコードを追加してください。<br>

+ `$ touch spec/support/vcr.rb`を実行<br>

+ `spec/support/vcr.rb`を編集(P196〜)<br>

```rb:vcr.rb
require "vcr"

VCR.configure do |config|
  config.cassette_library_dir = "#{::Rails.root}/spec/cassettes"
  config.hook_into :webmock
  config.ignore_localhost = true
  config.ignore_hosts 'chromedrinver.storage.googleapis.com'
  config.configure_rspec_metadata!
end
```

この新しい設定ファイルでは カセット(cassette) 、すなわち記録されたやりとりをアプリケーションの spec/cassettes ディレクトリに保存するように設定しています。<br>
すでに説明したとおり、スタブ化には WebMock を使います(ですが、VCR は他の数多くのスタブライブラリをサポートしています)。<br>
タスクを完了済みにする AJAX コールなどで使われる localhost へのリクエストは無視します。<br>
また、第6章でインストールした Webdrivers gem がインターネット上の ChromeDriver をダウンロードしに行く場合があるため、このアクセスも無視するようにしました。<br>
最後に、RSpec のタグによって VCR が有効になるようにします。JavaScript を実行するテストに js: true を付けたのと同じように、VCR を使うテストでは vcr: true を付けることで VCR が有効化されます。<br>
この設定が済んだら、新しいテストを実行して何が起きるか確認してください。<br>
テストの 実行はずっと速くなりますが、次のように失敗してしまいます。<br>

+ `$ bundle exec rspec`を実行(失敗する)<br>

```
Failures:

  1) User performs geocoding
     Failure/Error: user.geocode

     VCR::Errors::UnhandledHTTPRequestError:


       ================================================================================
       An HTTP request has been made that VCR does not know how to handle:
         GET http://ipinfo.io/161.185.207.20/geo

       There is currently no cassette in use. There are a few ways
       you can configure VCR to handle this request:
```

VCR が設定されていると、外部への HTTP リクエストは UnhandledHTTPRequestError という例外を起こして失敗するようになります。<br>
この問題を解消するために、vcr: true というオプションをテストに追加してください。<br>

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

  it { is_expected.to validate_presence_of :first_name }
  it { is_expected.to validate_presence_of :last_name }
  it { is_expected.to validate_presence_of :email }
  it { is_expected.to validate_uniqueness_of(:email).case_insensitive }

  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string" do
    user = FactoryBot.build(:user, first_name: "John", last_name: "Doe")
    expect(user.name).to eq "John Doe"
  end

  # アカウントが作成された時にウェルカムメールを送信すること
  it "sends a welcome email an account creation" do
    allow(UserMailer).to receive_message_chain(:welcome_email, :deliver_later)
    user = FactoryBot.create(:user)
    expect(UserMailer).to have_received(:welcome_email).with(user)
  end

  # ジオコーディングを実行すること
  it "performs geocoding", vcr: true do # 編集
    user = FactoryBot.create(:user, last_sign_in_ip: "161.185.207.20")
    expect {
      user.geocode
    }.to change(user, :location).
      from(nil).
      to("Brooklyn, New York, US")
  end
end
```

+ `$ spec/cassettes/User/performs_geocoding.yml`を編集<br>

```yml:performs_geocoding.yml
---
http_interactions:
  - request:
      method: get
      uri: http://ipinfo.io/161.185.207.20/geo
      body:
        encoding: US-ASCII
        string: ""
      headers:
        Accept-Encoding:
          - gzip;q=1.0,deflate;q=0.6,identity;q=0.3
        Accept:
          - "*/*"
        User-Agent:
          - Ruby
    response:
      status:
        code: 200
        message: OK
      headers:
        Date:
          - Wed, 11 Jul 2018 04:46:59 GMT
        Content-Type:
          - application/json; charset=utf-8
        Transfer-Encoding:
          - chunked
        Vary:
          - Accept-Encoding
        X-Powered-By:
          - Express
        X-Cloud-Trace-Context:
          - 72c73f8f3c50820effc90c4aabb84d58/8793541516014538142;o=0
        Access-Control-Allow-Origin:
          - "*"
        X-Content-Type-Options:
          - nosniff
        Via:
          - 1.1 google
      body:
        encoding: ASCII-8BIT
        string: |-
          {
            "ip": "161.185.207.20",
            "city": "Brooklyn",
            "region": "New York",
            "country": "US",
            "loc": "40.6944,-73.9906",
            "postal": "11201"
          }
      http_version:
    recorded_at: Wed, 11 Jul 2018 04:46:59 GMT
recorded_with: VCR 3.0.3
```

+ `$ bundle exec rspec`を実行(パスする)<br>

テストを再実行すると、二つの変化が起きるはずです。一点目はテストがパスするようになることです。<br>
二点目はリクエストとレスポンスの記録が、先ほど設定した spec/cassettes ディレクトリのファイルに保存されます。<br>
このファイルをちょっと覗いてみてください。このファイルは YAML ファイルになっていて、最初にリクエストのパラメータが記録され、そのあとにジオコーディングサービスから返ってきたパラメータが記録されています。<br>
テストをもう一度実行してください。依然としてテストはパスしますが、今回はカセットファイルの 内容を使って実際の HTTPリクエストとレスポンスをスタブ化します。<br>
この例ではモデルスペックで実行された HTTP トランザクションを記録するために VCR を使いましたが、テストの一部で HTTP トランザクションが発生するテストであれば、どのレイヤーのどのテストでも VCR を利用することができます。<br>
私は VCR が大好きですが、VCR には短所もあります。特に、カセットが古びてしまう問題には注意が必要です。<br>
これはつまり、もしテストに使っている外部 API の仕様が変わってしまっても、あなたはカセットが古くなっていることを知る術がない、ということです。<br>
それを知ることができる唯一の方法は、カセットを削除して、もう一度テストを実行することです。Railsの開発者の多くはプロジェクトのバージョン管理システムからカセットファイルを除外することを選択します。<br>
これは新しい開発者が最初にテストスイートを実行した際に、必ず自分でカセットを記録するようにするためです。<br>
この方法を採用する前に、[一定の頻度でカセットを自動的に再記録する方法](https://relishapp.com/vcr/vcr/v/3-0-3/docs/cassettes/automatic-re-recording)を検討しても良いかもしれません。<br>
この方法を使えば、後方互換性のない API の変更を比較的早く検知することができます。<br>
また、二つ目の注意点として、API のシークレットトークンやユーザーの個人情報といった機密情報をカセットに含めないようにしてください。<br>
VCR には[サニタイズ用のオプション](https://relishapp.com/vcr/vcr/v/3-0-3/docs/configuration/filter-sensitive-data)が用意されているので、これを使えば機密情報がファイルに保存される問題を回避することができます。<br>
もし、みなさんがカセットをバージョン管理システムにあえて保存するのであれば、この点は非常に重要です。<br>

## まとめ

メールやファイルアップロード、web サービス、バックグラウンドプロセスといった機能は、あなたのアプリケーションの中では些細な機能かもしれません。<br>
しかし、必要に応じてその機能をテストする時間も作ってください。なぜなら、この先何が起こるかわからないからです。<br>
そのwebサービスがあるときからアプリケーションの重要な機能になるかもしれないですし、あなたが次に作るアプリケーションがメールを多用するものになるかもしれません。<br>
練習を繰り返す時間が無駄になることは決してないのです。<br>
これであなたは私が普段テストするときのノウハウを身につけました。必ずしも全部がエレガントな方法だとは限りませんが、結果としては十分なカバレッジを出せています。<br>
そして、そのおかげで私は気軽に機能を追加できています。既存の機能を壊してしまう心配はいりません。<br>
万一 、何かを壊してしまったとしても、私はテストを使ってその場所を特定し、内容に応じて問題を修復することができます。<br>
RSpec と Rails に関する説明はそろそろ終わりに近づいてきたので、次は今までに得た知識を使い、もっとテスト駆動らしくソフトウェアを開発する方法についてお話ししたいと思います。これが次章で説明する内容です。<br>

## 演習問題


+ もしあなたのアプリケーションにメール送信機能があるなら、練習としてその機能をテ ストしてください。<br>
  候補としてよく挙がりそうなのはパスワードリセットのメッセージと通知かもしれません。<br>
  もしそうした機能がないのであれば、本書のサンプルアプリケーションの中にある、Devise のパスワードリセット機能をテストしてみてください。<br>

+ あなたのアプリケーションにはファイルアップロード機能や、バックグラウンドジョブはありますか? 繰り返しますが、練習用にこうした機能をテストするのは非常に良い考えです。<br>
  テストをするときは本章で紹介したユーティリティも使ってください。こうした機能はよく忘れ去られます。<br>
  そして早朝や深夜に止まって初めて思い出されるのです。<br>

+ あなたは外部の認証サービスや支払い処理、その他のwebサービス用のスペックを書いたことがありますか? VCR を使ってスピードアップするにはどうすればよいですか?<br>
