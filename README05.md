## ファクトリ内の重複をなくす

```
Factory Bot では同じ型を作成するファクトリを複数定義することもできます。
たとえば、スケジュールどおりのプロジェクトとスケジュールから遅れているプロジェクトをテストしたいのであれば、別々の名前を付けてプロジェクトファクトリの引数に渡すことができます。
その際はそのファクトリを使って作成するインスタンスのクラス名と、既存のファクトリと異なるインスタンスの属性値(この例でいうと due_on 属性の値)も指定します。
```

+ `spec/factories/projects.rb`を編集<br>

```rb:projects.rb
FactoryBot.define do
  factory :project do
    sequence(:name) { |n| "Project #{n}" }
    description { "A test project." }
    due_on { 1.week.from_now }
    association :owner
  end

  # 追加
  # 昨日が締め切りのプロジェクト
  factory :project_due_yesterday, class: Project do
    sequence(:name) { |n| "Test Project #{n}" }
    description { "Sample project for testing purposes" }
    due_on { 1.day.ago }
    association :owner
  end

  # 今日が締め切りのプロジェクト
  factory :project_due_today, class: Project do
    sequence(:name) { |n| "Test Project #{n}" }
    description { "Sample project for testing purposes" }
    due_on { Date.current.in_time_zone }
    association :owner
  end

  # 明日が締め切りのプロジェクト
  factory :project_due_tomorrow, class: Project do
    sequence(:name) { |n| "Test Project #{n}" }
    description { "Sample project for testing purposes" }
    due_on { 1.day.from_now }
    association :owner
  end
  # ここまで
end
```

```
こうすると上で定義した新しいファクトリを Project モデルのスペックで使うことができます。
ここでは魔法のマッチャ、be_late が登場します。be_late は RSpec に定義されているマッチャではありません。
ですが RSpec は賢いので、project に late または late? という名前の属性やメソッドが存在し、
それが真偽値を返すようになっていれば be_late はメソッドや 属性の戻り値が true になっていることを検証してくれるのです。
すごいですね。
```

+ `spec/models/project_spec.rb`を編集<br>

```rb:project_spec.rb
require 'rails_helper'

RSpec.describe Project, type: :model do
  # ユーザー単位では重複したプロジェクト名を許可しないこと
  it "does not allow duplicate project names per user" do
    user = User.create(
      first_name: "Joe",
      last_name: "Tester",
      email: "joetester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )

    user.projects.create(
      name: "Test Project",
    )

    new_project = user.projects.build(
      name: "Test Project",
    )

    new_project.valid?
    expect(new_project.errors[:name]).to include("has already been taken")
  end

  # 二人のユーザーが同じ名前を使うことは許可すること
  it "allows two user to share a project name" do
    user = User.create(
      first_name: "Joe",
      last_name: "Tester",
      email: "joetester@examle.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )

    user.projects.create(
      name: "Test Project",
    )

    other_user = User.create(
      first_name: "Jane",
      last_name: "Tester",
      email: "janetester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )

    other_project = other_user.projects.build(
      name: "Test Project",
    )

    expect(other_project).to be_valid
  end

  # 追加
  # 遅延ステータス
  describe "late status" do
    # 締め切り日が過ぎていれば遅延していること
    it "is late when the due date is past today" do
      project = FactoryBot.create(:project_due_yesterday)
      expect(project).to be_late
    end

    # 締め切り日が今日ならスケジュールどおりであること
    it "is on time when the due date is today" do
      project = FactoryBot.create(:project_due_today)
      expect(project).to_not be_late
    end

    # 締め切り日が未来ならスケジュールどおりであること
    it "is on time when the due date is in the future" do
      project = FactoryBot.create(:project_due_tomorrow)
      expect(project).to_not be_late
    end
  end
  # ここまで
end
```

```
ですが、新しく作ったファクトリには大量の重複があります。
新しいファクトリを定義するときは毎回プロジェクトの全属性を再定義しなければいけません。
これはつまり、Project モデルの属性を変更したときは毎回複数のファクトリ定義を変更する必要が出てくる、ということを意味しています。
Factory Bot には重複を減らすテクニックが二つあります。
一つ目は ファクトリの継承を使ってユニークな属性だけを変えることです。
```

+ `spec/factories/projects.rb`を編集<br>

```rb:projects.rb
FactoryBot.define do
  # 編集
  factory :project do
    sequence(:name) { |n| "Test Project #{n}" }
    description { "Sample project for testing purposes" }
    due_on { 1.week.from_now }
    association :owner

    # 昨日が締め切りのプロジェクト
    factory :project_due_yesterday do
      due_on { 1.day.ago }
    end

    # 今日が締め切りのプロジェクト
    factory :project_due_today do
      due_on { Date.current.in_time_zone }
    end

    # 明日が締め切りのプロジェクト
    factory :project_due_tomorrow do
      due_on { 1.day.from_now }
    end
  end
  # ここまで
end
```

```
yesterday と :project_due_today と :project_due_tomorrow の各ファクトリは継承元となる :project ファクトリの内部で入れ子になっています。
構造だけを抜き出すと次のようになります。

factory :project
  factory :project_due_yesterday
  factory :project_due_today
  factory :project_due_tomorrow

継承を使うと class: Project の指定もなくすことができます。
なぜならこの構造から Factory Bot は子ファクトリで Project クラスを使うことがわかるからです。
この場合、スペック側は何も変更しなくてもそのままでパスします。
重複を減らすための二つ目のテクニックは トレイト(trait) を使ってテストデータを構築することです。
このアプローチでは属性値の集合をファクトリで定義します。まず、プロジェクトファクトリの中身を更新しましょう。
```

+ `spec/factories/projects.rb`を編集<br>

```rb:projects.rb
FactoryBot.define do
  factory :project do
    sequence(:name) { |n| "Test Project #{n}" }
    description { "Sample project for testing purposes" }
    due_on { 1.week.from_now }
    association :owner

    # 編集
    # 締め切りが昨日
    trait :due_yesterday do
      due_on { 1.day.ago }
    end

    # 締め切りが今日
    trait :due_today do
      due_on { Date.current.in_time_zone }
    end

    # 締め切りが明日
    trait :due_tomorrow do
      due_on { 1.day.from_now }
    end
    # ここまで
  end
end
```

```
トレイトを使うためにはスペックを変更する必要があります。
利用したいトレイトを使って次のようにファクトリから新しいプロジェクトを作成してください。
```

+ `spec/models/project_spec.rb`を編集<br>

```rb:project_spec.rb
require 'rails_helper'

RSpec.describe Project, type: :model do
  # ユーザー単位では重複したプロジェクト名を許可しないこと
  it "does not allow duplicate project names per user" do
    user = User.create(
      first_name: "Joe",
      last_name: "Tester",
      email: "joetester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )

    user.projects.create(
      name: "Test Project",
    )

    new_project = user.projects.build(
      name: "Test Project",
    )

    new_project.valid?
    expect(new_project.errors[:name]).to include("has already been taken")
  end

  # 二人のユーザーが同じ名前を使うことは許可すること
  it "allows two user to share a project name" do
    user = User.create(
      first_name: "Joe",
      last_name: "Tester",
      email: "joetester@examle.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )

    user.projects.create(
      name: "Test Project",
    )

    other_user = User.create(
      first_name: "Jane",
      last_name: "Tester",
      email: "janetester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )

    other_project = other_user.projects.build(
      name: "Test Project",
    )

    expect(other_project).to be_valid
  end

  # 遅延ステータス
  describe "late status" do
    # 締め切り日が過ぎていれば遅延していること
    it "is late when the due date is past today" do
      project = FactoryBot.create(:project, :due_yesterday) # 編集
      expect(project).to be_late
    end

    # 締め切り日が今日ならスケジュールどおりであること
    it "is on time when the due date is today" do
      project = FactoryBot.create(:project, :due_today) # 編集
      expect(project).to_not be_late
    end

    # 締め切り日が未来ならスケジュールどおりであること
    it "is on time when the due date is in the future" do
      project = FactoryBot.create(:project, :due_tomorrow) # 編集
      expect(project).to_not be_late
    end
  end
end
```

+ `bundle exec rspec`を実行<br>

```
トレイトを使うことの本当の利点は、複数のトレイトを組み合わせて複雑なオブジェクトを構築できる点です。
トレイトについてはこの後の章でテストデータに関する要件がもっと複雑になってきたときに再度説明します。
```

## コールバック

```
Factory Bot の機能をもうひとつ紹介しましょう。
コールバックを使うと、ファクトリがオ ブジェクトを create する前、もしくは create した後に何かしら追加のアクションを実行できます。
また、create されたときだけでなく、build されたり、stub されたりしたときも同じように使えます。
適切にコールバックを使えば複雑なテストシナリオも簡単にセットアップできるので、強力な時間の節約になります。
ですが、一方でコールバックは遅いテストや無駄 に複雑なテストの原因になることもあります。注意して使ってください。
そのことを頭の片隅に置きつつ、コールバックのよくある使い方を見てみましょう。
ここでは複雑な関連を持つオブジェクトを作成する方法を説明します。
Factory Bot にはこうした 処理を簡単に行うための create_list メソッドが用意されています。
コールバックを利用して、新しいオブジェクトが作成されたら自動的に複数のメモを作成する処理を追加してみましょう。
今回は必要なときにだけコールバックを利用するよう、トレイトの中でコールバックを使います。
```

+ `spec/factories/projects.rb`を編集<br>

```rb:projects.rb
FactoryBot.define do
  factory :project do
    sequence(:name) { |n| "Test Project #{n}" }
    description { "Sample project for testing purposes" }
    due_on { 1.week.from_now }
    association :owner

    # 追加
    # メモ付きのプロジェクト
    trait :with_notes do
      after(:create) { |project| create_list(:note, 5, project: project) }
    end
    # ここまで

    # 締め切りが昨日
    trait :due_yesterday do
      due_on { 1.day.ago }
    end

    # 締め切りが今日
    trait :due_today do
      due_on { Date.current.in_time_zone }
    end

    # 締め切りが明日
    trait :due_tomorrow do
      due_on { 1.day.from_now }
    end
  end
end
```

```
create_list メソッドではモデルを作成するために関連するモデルが必要です。
今回はメモの作成に必要な Project モデルを使っています。
プロジェクトファクトリに新しく定義した with_notes トレイトは、新しいプロジェクトを作成した後にメモファクトリを使って5つの新しいメモを追加します。
それではスペック内でこのトレイトを使う方法を見てみましょう。最初はトレイトなしのファクトリを使ってみます。
```

+ `spec/models/project_spec.rb`を編集<br>

```rb:project_spec.rb
require 'rails_helper'

RSpec.describe Project, type: :model do
  # ユーザー単位では重複したプロジェクト名を許可しないこと
  it "does not allow duplicate project names per user" do
    user = User.create(
      first_name: "Joe",
      last_name: "Tester",
      email: "joetester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )

    user.projects.create(
      name: "Test Project",
    )

    new_project = user.projects.build(
      name: "Test Project",
    )

    new_project.valid?
    expect(new_project.errors[:name]).to include("has already been taken")
  end

  # 二人のユーザーが同じ名前を使うことは許可すること
  it "allows two user to share a project name" do
    user = User.create(
      first_name: "Joe",
      last_name: "Tester",
      email: "joetester@examle.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )

    user.projects.create(
      name: "Test Project",
    )

    other_user = User.create(
      first_name: "Jane",
      last_name: "Tester",
      email: "janetester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )

    other_project = other_user.projects.build(
      name: "Test Project",
    )

    expect(other_project).to be_valid
  end

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

  # 追加
  it "can have many notes" do
    project = FactoryBot.create(:project)
    expect(project.notes.length).to eq 5
  end
  # ここまで
end
```

```
このテストは失敗します。なぜならメモの数が5件ではなくゼロだからです。

Failures:

  1) Project can have many notes
     Failure/Error: expect(project.notes.length).to eq 5

       expected: 5
            got: 0

       (compared using ==)
     # ./spec/models/project_spec.rb:75:in `block (2 levels) in <main>'

Finished in 0.68483 seconds (files took 2.35 seconds to load)
18 examples, 1 failure

Failed examples:

rspec ./spec/models/project_spec.rb:73 # Project can have many notes
```

そこで `with_notes` トレイトでセットアップした新しいコールバックを使って、このテストをパスさせましょう。<br>

+ `spec/models/project_spec.rb`を編集<br>

```rb:project_spec.rb
require 'rails_helper'

RSpec.describe Project, type: :model do
  # ユーザー単位では重複したプロジェクト名を許可しないこと
  it "does not allow duplicate project names per user" do
    user = User.create(
      first_name: "Joe",
      last_name: "Tester",
      email: "joetester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )

    user.projects.create(
      name: "Test Project",
    )

    new_project = user.projects.build(
      name: "Test Project",
    )

    new_project.valid?
    expect(new_project.errors[:name]).to include("has already been taken")
  end

  # 二人のユーザーが同じ名前を使うことは許可すること
  it "allows two user to share a project name" do
    user = User.create(
      first_name: "Joe",
      last_name: "Tester",
      email: "joetester@examle.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )

    user.projects.create(
      name: "Test Project",
    )

    other_user = User.create(
      first_name: "Jane",
      last_name: "Tester",
      email: "janetester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )

    other_project = other_user.projects.build(
      name: "Test Project",
    )

    expect(other_project).to be_valid
  end

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
    project = FactoryBot.create(:project, :with_notes) # 編集
    expect(project.notes.length).to eq 5
  end
end
```

+ `$ bundle exec rspec`を実行<br>

```
これでテストがパスします。なぜなら、コールバックによってプロジェクトに関連する5つのメモが作成されるからです。
実際のアプリケーションでこういう仕組みを使っていると、ちょっと情報量の乏しいテストに見えるかもしれません。
ですが、今回の使用例はコールバックが正しく設定されているか確認するのに役立ちますし、この先でもっと複雑なテストを作り始める前のちょうどいい練習にもなります。
とくに、Rails のモデルが入れ子になった他のモデルを属性として持っている場合、コールバックはそうしたモデルのテストデータを作るのに便利です。
```

`
ここでは Factory Bot のコールバックについてごく簡単な内容しか説明していません。<br>
コールバックの詳しい使い方についてはFactory Bot の[公式ドキュメントにあるコールバックの欄29](https://github.com/thoughtbot/factory_bot/blob/master/GETTING_STARTED.md%23callbacks)を参照してください。<br>
本書でもこのあとでさらにコールバックを使っていきます。
`

## ファクトリを安全に使うには

```
ファクトリはテスト駆動開発で強力なパワーを発揮します。
ほんの数行コードを書くだけで、私たちのソフトウェアを検証するために必要なサンプルデータをあっという間に作ってくれます。
セットアップ用のコードも短くなるので、テストコードの読みやすさも妨げません。

ですが、他のパワフルなツールと同様に、ファクトリを使うときにはちょっと注意した方がいいと忠告しておきます。
前述のとおり、ファクトリを使うとテスト中に予期しないデータが作成されたり、無駄にテストが遅くなったりする原因になります。
上記のような問題がテストで発生した場合はまず、ファクトリが必要なことだけを行い、それ以上のことをやっていないことを確認してください。
コールバックを使って関連するデータを作成する必要があるなら、ファクトリを使うたびに呼び出されないよう、トレイトの中でセットアップするようにしましょう。
可能な限り FactoryBot.create よりも FactoryBot.build を使ってください。
こうすればテストデータベースにデータを追加する回数が減るので、パフォーマンス面のコストを削減できます。

最初の頃は ここでファクトリを使う必要はあるだろうか? と自問するのもいいでしょう。
もしモデルの new メソッドや create メソッドでテストデータをセットアップできるなら、ファクトリをまったく使わずに済ませることもできます。
PORO で作ったデータとファクトリで作ったデータをテスト内に混在させることもできます。
このように、テストの読みやすさ と速さを保つためにはいろんな方法が使えます。
```

```
テストの経験がある程度ある人であれば、テストのスピードを上げる方法として、なぜ モック や ダブル を説明しないのか、と不思議に思っている人もいるかもしれません。
モックを使うとさまざまな種類の複雑さをテストに持ち込むことになります。
なので、初心者のうちはここで説明したような方法でデータを作る方が良いと私は考えています。
モックやスタブについては本書の後半で説明します。
```

## まとめ

`
この章では Factory Bot を使ってごちゃごちゃしていたスペックをきれいにしました。 <br>
それだけでなく、データを作成する際の柔軟性も上がり、リアルなシナリオをテストしやすくなりました。 <br>
この章で説明した機能は私が普段よく使う機能です。こうした機能はみなさんも大半のテストで使うことになるはずです。 <br>
また、ここで紹介した内容と同じぐらい、たくさんの機能が Factory Bot には用意されています。 <br>
Factory Bot の基礎を理解したら、ぜひ[Factory Bot のドキュメント30](https://github.com/thoughtbot/factory_bot/blob/master/GETTING_STARTED.md)を読んで、たくさんの便利機能を使いこなせるようになってください。 <br>
本書ではこのあとの章でも Factory Bot を使っていきます。 <br>
実際、Factory Bot はこの次に出てくるテストコードでも重要な役割を演じます。 <br>
さて、次に出てくるコードはコントローラです。 <br>
コントローラはモデルとビューの間でデータをやりとりするためのコンポーネントです。そしてこのコントローラが次章のメイントピックになります。 <br>
`

## 演習問題

• 第3章で追加したモデルスペックをもう一度確認してください。ファクトリを使ってきれいにできる箇所が他にもありませんか
ちょっとヒントを出しましょう。<br>
現状の spec/models/project_spec.rb では User オブジェクトを作成するコードがたくさんあります。<br>
かわりにユーザーファクトリを使って書きかえることはできますか?<br>

• もしあなた自身のアプリケーションにまだファクトリを追加していなければ、ファクトリを追加してください。<br>

• あなたが作ったアプリケーションのファクトリを見てください。<br>
継承やトレイトを使ってファクトリをリファクタリングできませんか?<br>

• コールバック付きのトレイトを作成してみてください。<br>
それから簡単なテストを書いて期待どおりに動いているか確認してください。<br>
