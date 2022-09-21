## `project_spec.rb` の作成

```
続いてもっと複雑なバリデーションをテストしましょう。User モデルの話はいったん横に 置いて、今度は Project モデルに着目します。たとえば、ユーザーは同じ名前のプロジェクト を作成できないという要件があったとします。つまり、プロジェクト名はユーザーごとにユ ニークでなければならない、ということです。別の言い方をすると、私は Paint the house (家 を塗る)という複数のプロジェクトを持つことはできないが、あなたと私はそれぞれ Paint the house というプロジェクトを持つことができる、ということです。あなたならどうやって テストしますか?
では Project モデル用に新しいスペックファイルを作成しましょう。
```

+ `$ bin/rails g rspec:model project`を実行<br>

```
続いて、作成されたファイルに二つの example を追加します。ここでテストしたいのは、 一人のユーザーは同じ名前で二つのプロジェクトを作成できないが、ユーザーが異なるとき は同じ名前のプロジェクトを作成できる、という要件です。
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
end
```

+ projectモデルには以下のようなバリデーションが設定されている<br>

+ 下記のバリデーションコードを消したり付けたりしてテストを試してみる

```rb:project.rb
validates :name, presence: true, uniqueness: { scope :user_id }
```

+ `$ bundle exec rspec`を実行<br>

```
Project
  does not allow duplicate project names per user
  allows two user to share a project name

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


Finished in 0.34779 seconds (files took 2.35 seconds to load)
8 examples, 0 failures, 2 pending
```

## インスタンスメソッドをテストする

```
それでは User モデルのテストに戻ります。このサンプルアプリケーションでは、ユーザー の姓と名を毎回連結して新しい文字列を作るより、@user.name を呼び出すだけでフルネーム が出力されるようにした方が便利です。というわけでこんなメソッドが User クラスに作っ てあります。
```

+ `app/models/user.rb`<br>

```rb:user.rb
def name
  [first_name, last_name].join(' ')
end
```

バリデーションの example と同じ基本的なテクニックでこの機能の exampleを作ることができます。

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

  # ユーザーのフルネームを文字列として返すこと
  # 追加
  it "returns a user's full name as a string" do
    user = User.new(
      first_name: 'John',
      last_name: "Doe",
      email: "johndoe@example.com",
    )
    expect(user.name).to eq "John Doe"
  end
  # ここまで
end
```

※ RSpecで等値のエクスペクテーションを書くときは `==` ではなく `eq` を使います。<br>


## クラスメソッドとスコープをテストする

```
このアプリケーションには渡された文字列でメモ(note)を検索する機能を用意してあり ます。念のため説明しておくと、この機能は Note モデルにスコープとして実装されていま す。
```

+ `app/models/note.rb`<br>

```rb:note.rb
scope :search, ->(term) {
  where("LOWER(message) LIKE ?", "%#{term.downcase}%")
}
```

+ `$ bin/rails g rspec:model project`を実行<br>

+ `spec/models/note_spec.rb`を編集<br>

```rb:note_spec.rb
require 'rails_helper'

RSpec.describe Note, type: :model do
  # 検索文字列に一致するメモを返すこと
  it "returns notes that match the search term" do
    user = User.create(
      first_name: 'Joe',
      last_name: 'Tester',
      email:     'joetester@example.com',
      password:  'dottle-nouveau-pavilion-tights-furze',
    )

    project = user.projects.create(
      name: "Test Project"
    )

    note1 = project.notes.create(
      message: "This is the first note.",
      user: user,
    )
    note2 = project.notes.create(
      message: "This is the second note.",
      user: user,
    )
    note3 = project.notes.create(
      message: "First, preheat the oven",
      user: user,
    )

    expect(Note.search("first")).to include(note1, note3)
    expect(Note.search("first")).to_not include(note2)
  end
end
```

```
search スコープは検索文字列に一致するメモのコレクションを返します。
返されたコレクションは一致したメモだけが含まれるはずです。
その文字列を含まないメモはコレクションに含まれません。
```

+ `$ bundle exec rspec`を実行<br>

```
Note
  returns notes that match the search term

Project
  does not allow duplicate project names per user
  allows two user to share a project name

User
  is valid with a first name, last name, email, and password
  is invalid without a first name
  is invalid without a last name
  is invalid without an email address (PENDING: Not yet implemented)
  is invalid with a duplicate email address
  returns a user's full name as a string

Pending: (Failures listed here are expected and do not affect your suite's status)

  1) User is invalid without an email address
     # Not yet implemented
     # ./spec/models/user_spec.rb:30


Finished in 0.30389 seconds (files took 2.39 seconds to load)
9 examples, 0 failures, 1 pending
```

このテストでは次のような実験ができます。`to` を `to_not` に変えたらどうなるでしょうか?<br>
もしくは検索文字列を含むメモをさらに追加したらどうなるでしょうか?<br>


## 失敗をテストする

```
正常系のテストは終わりました。ユーザーが文字列検索すると結果が返ってきます。
しかし、結果が返ってこない文字列で検索したときはどうでしょうか?
そんな場合もテストした方が良いです。次のスペックがそのテストになります。
```

+ `spec/models/note_spec.rb`を編集<br>

```rb:note_spec.rb
require 'rails_helper'

RSpec.describe Note, type: :model do
  # 検索文字列に一致するメモを返すこと
  it "returns notes that match the search term" do
    user = User.create(
      first_name: 'Joe',
      last_name: 'Tester',
      email:     'joetester@example.com',
      password:  'dottle-nouveau-pavilion-tights-furze',
    )

    project = user.projects.create(
      name: "Test Project"
    )

    note1 = project.notes.create(
      message: "This is the first note.",
      user: user,
    )
    note2 = project.notes.create(
      message: "This is the second note.",
      user: user,
    )
    note3 = project.notes.create(
      message: "First, preheat the oven",
      user: user,
    )

    expect(Note.search("first")).to include(note1, note3)
    expect(Note.search("first")).to_not include(note2)
  end

  # 追加
  # 検索結果が一件も見つからなければ空のコレクションを返すこと
  it "returns an empty collection when no results are found" do
    user = User.create(
      first_name: "Joe",
      last_name: "Tester",
      email:     "joetster@example.com",
      password:  "dottle-nouveau-pavilion-tights-furze",
    )

    project = user.projects.create(
      name: "Test Project",
    )

    note1 = project.notes.create(
      message: "This is the first note.",
      user: user,
    )
    note2 = project.notes.create(
      message: "This is the second note.",
      user: user,
    )
    note3 = project.notes.create(
      message: "First, preheat the oven.",
      user: user,
    )

    expect(Note.search("message")).to be_empty
  end
  # ここまで
end
```

```
このスペックでは Note.search("message") を実行して返却された配列をチェックします。
この配列は確かに空なのでスペックはパスします!
これで理想的な結果、すなわち結果が返ってくる文字列で検索した場合だけでなく、
結果が返ってこない文字列で検索した場合も テストしたことになります。
```

## マッチャについてもっと詳しく

```
これまで四つのマッチャ(be_valid 、eq 、include 、be_empty )を実際に使いながら見てきました。
最初に使ったのは be_valid です。このマッチャは rspec-rails gem が提供するマッチャで、Rails のモデルの有効性をテストします。
eq と include は rspec-expectations で定義されているマッチャで、前章で RSpec をセットアップしたときに rspec-rails と一緒にインストールされました。
```

RSpec が提供するデフォルトのマッチャをすべて見たい場合はGitHubにある [rspec- expectations リポジトリ23](https://github.com/rspec/rspec-expectations) の README が参考になるかもしれません。<br>
この中に出てくるマッチ ャのいくつかは本書全体を通して説明していきます。<br>
また、第8章では自分でカスタムマッチャを作る方法も説明します。<br>

## describe、context、before、after を使ってスペックを DRYにする

```
ここまでに作成したメモ用のスペックには冗⻑なコードが含まれます。
具体的には、各 example の中ではまったく同じ4つのオブジェクトを作成しています。
アプリケーションコードと同様に、DRY 原則はテストコードにも当てはまります(いくつか例外もあるので、のちほど説明します)。
では RSpec の機能をさらに活用してテストコードをきれいにしてみましょう。
```

```
先ほど作った Note モデルのスペックに注目してみましょう。
まず最初にやるべきことは describe ブロックを describe Note ブロックの中に作成することです。
これは検索機能に フォーカスするためです。アウトラインを抜き出すと、このようになります。
```

+ `spec/models/note_spec.rb`を編集<br>

```rb:note_spec.rb

RSpec.describe Note, type: :model do
  # バリデーション用のスペックが並ぶ

  # 文字列に一致するメッセージを検索する
  describe "search message for a term" do
    # 検索用の examle が並ぶ ...
  end
end
```

```
二つの context ブロックを加えてさらに example を切り分けましょう。
一つは「一致する データが見つかるとき」で、もう一つは「一致するデータが1件も見つからないとき」です。
```

+ `spec/models/note_spec.rb`を編集<br>

```rb:note_spec.rb
require 'rails_helper'

RSpec.describe Note, type: :model do
  # 他のスペックが並ぶ

  # 文字列に一致するメッセージを検索する
  describe "search message for a term" do

    # 一致するデータが見つかるとき
    context "when a match is found" do
      # 一致する場合の examle が並ぶ ...
    end

    # 一致するデータが1件も見つからないとき
    context "when no match is found" do
      # 一致しない場合の example が並ぶ ...
    end
  end
end
```

```
describe と context は技術的には交換可能なのですが、私は次のように使い分ける のが好きです。
すなわち、describe ではクラスやシステムの機能に関するアウトラ インを記述し、context では特定の状態に関するアウトラインを記述するようにしま す。
このケースであれば、状態は二つあります。一つは結果が返ってくる検索文字列 を渡された状態で、もう一つは結果が返ってこない検索文字列が渡された状態です。
```

```
お気づきかもしれませんが、このように example のアウトラインを作ると、同じような example をひとまとめにして分類できます。
こうするとスペックがさらに読みやすくなります。
では最後に、before フックを利用してスペックのリファクタリングを完了させましょう。
before ブロックの中に書かれたコードは内側の各テストが実行される前に実行されます。
また、before ブロックは describe や context ブロックによってスコープが限定されます。
たとえばこの例で言うと、before ブロックのコードは "search message for a term" ブロックの内側にある全部のテストに先立って実行されます。
ですが、新しく作った describe ブロッ クの外側にあるその他の example の前には実行されません。
```

+ `spec/models/note_spec.rb`を編集<br>

```rb:note_spec.rb
require 'rails_helper'

RSpec.describe Note, type: :model do
  before do
    # このファイルの全てストで使用するテストデータをセットアップする
  end

  # バリデーションのテストが並ぶ

  # 文字列に一致するメッセージを検索する
  describe "search message for a term" do

    before do
      # 検索機能の全テストに関連する追加のテストデータをセットアップする
    end

    # 一致するデータが見つかるとき
    context "when a match is found" do
      # 一致する場合の examle が並ぶ ...
    end

    # 一致するデータが1件も見つからないとき
    context "when no match is found" do
      # 一致しない場合の example が並ぶ ...
    end
  end
end
```

```
RSpec の before フックはスペック内の冗⻑なコードを認識し、きれいにするための良い出 発点になります。
これ以外にも冗⻑なテストコードをきれいにするテクニックはありますが、 before を使うのが最も一般的かもしれません。
before ブロックは example ごとに、またはブ ロック内の各 example ごとに、またはテストスイート全体を実行するごとに実行されます。
```

```
• before(:each) は describe または context ブロック内の 各(each) テストの前に実行 されます。好みに応じて before(:example) というエイリアスを使ってもいいですし、 上のサンプルコードで書いたように before だけでも構いません。
もしブロック内に4つ のテストがあれば、before のコードも4回実行されます。

• before(:all) は describe または context ブロック内の 全(all) テストの前に一回だ け実行されます。
かわりに before(:context) というエイリアスを使っても構いません。
こちらは before のコードは一回だけ実行され、それから4つのテストが実行されます。

• before(:suite) はテストスイート全体の全ファイルを実行する前に実行されます。
before(:all) と before(:suite) は時間のかかる独立したセットアップ処理を1回だけ実行し、
テスト全体の実行時間を短くするのに役立ちます。ですが、この機能を使うとテスト全 体を汚染してしまう原因にもなりかねません。
可能な限り before(:each) を使うようにしてください。
```

```
上で示したような書き方で before ブロックを定義すると、各(each) テストの前に ブロック内のコードが実行されます。
before のかわりに before :each のように定義 すれば、より明示的な書き方になります。
どちらを使っても構わないので、あなた 自身やあなたのチームの好みに応じてお好きな方を使ってください。
```

```
もし example の実行後に後片付けが必要になるのであれば(たとえば外部サービスとの接 続を切断する場合など)、after フックを使って各 example のあと(after)に後片付けすること もできます。
before と同様、after にも each 、all 、suite のオプションがあります。
RSpecの場合、デフォルトでデータベースの後片付けをやってくれるので、私は after を使うこと はほとんどありません。
さて、整理後の全スペックを見てみましょう。
```

+ `spec/models/note_spec.rb`を編集<br>

```rb:note_spec.rb
require 'rails_helper'

RSpec.describe Note, type: :model do
  before do
    # このファイルの全てストで使用するテストデータをセットアップする
    @user = User.create(
      first_name: "Joe",
      last_name: "Tester",
      email:     "joetester@example.com",
      password: "dottle-nouveau-pavilion-tights-furze",
    )

    @project = @user.projects.create(
      name: "Test Project",
    )
  end

  # ユーザー、プロジェクト、メッセージがあれば有効な状態であること
  it "is valid a user, project, and message" do
    note = Note.new(
      message: "This is a same note.",
      user: @user,
      project: @project
    )
    expect(note).to be_valid
  end

  # メッセージがなければ無効な状態であること
  it "is invalid without a message" do
    note = Note.new(message: nil)
    note.valid?
    expect(note.errors[:message]).to include("can't be blank")
  end

  # バリデーションのテストが並ぶ

  # 文字列に一致するメッセージを検索する
  describe "search message for a term" do

    before do
      # 検索機能の全テストに関連する追加のテストデータをセットアップする
      @note1 = @project.notes.create(
        message: "This is the first note.",
        user: @user
      )
      @note2 = @project.notes.create(
        message: "This is the second note.",
        user: @user,
      )
      @note3 = @project.notes.create(
        message: "First, preheat the oven.",
        user: @user,
      )
    end

    # 一致するデータが見つかるとき
    context "when a match is found" do
      # 一致する場合の examle が並ぶ ...
      # 文字列に一致するメモを返すこと
      it "returns notes that match the search term" do
        expect(Note.search("first")).to include(@note1, @note3)
      end
    end

    # 一致するデータが1件も見つからないとき
    context "when no match is found" do
      # 一致しない場合の example が並ぶ ...
      # 空のコレクションを返すこと
      it "returns an empty collection" do
        expect(Note.search("message")).to be_empty
      end
    end
  end
end
```

+ `$ bundle exec rspec`を実行<br>

```
Note
  is valid a user, project, and message
  is invalid without a message
  search message for a term
    when a match is found
      returns notes that match the search term
    when no match is found
      returns an empty collection

Project
  does not allow duplicate project names per user
  allows two user to share a project name

User
  is valid with a first name, last name, email, and password
  is invalid without a first name
  is invalid without a last name
  is invalid without an email address
  is invalid with a duplicate email address
  returns a user's full name as a string

Finished in 0.4454 seconds (files took 2.29 seconds to load)
12 examples, 0 failures
```

```
みなさんはもしかするとテストデータのセットアップ方法が少し変わったことに気づいたかもしれません。
セットアップの処理を各テストから before ブロックに移動したので、各ユーザーはインスタンス変数にアサインする必要があります。
そうしないとテストの中で変数名を指定してデータにアクセスできないからです。
これらのスペックを実行すると、こんなふうに素敵なアウトラインが表示されます(第2章でドキュメント形式を使うように RSpec を設定したからです)。
```

### まとめ

```
本章ではモデルのテストにフォーカスしましたが、このあとに登場するモデル以外のスペ ックでも使えるその他の重要なテクニックもたくさん説明しました。
• 期待する結果は能動形で明示的に記述すること。 example の結果がどうなるかを動詞を 使って説明してください。
チェックする結果は example 一つに付き一個だけにしてください。

• 起きてほしいことと、起きてほしくないことをテストすること。
example を書くときは両方のパスを考え、その考えに沿ってテストを書いてください。

• 境界値テストをすること。もしパスワードのバリデーションが4文字以上10文字以下なら、8文字のパスワードをテストしただけで満足しないでください。
4文字と10文字、そして3文字と11文字もテストするのが良いテストケースです。(もちろん、なぜそんなに 短いパスワードを許容し、なぜそれ以上⻑いパスワードを許容しないのか、と自問する チャンスかもしれません。
テストはアプリケーションの要件とコードを熟考するための 良い機会でもあります。)

• 可読性を上げるためにスペックを整理すること。describeとcontextはよく似た example を分類してアウトライン化します。
before ブロックと after ブロックは重複を 取り除きます。
しかし、テストの場合は DRY であることよりも読みやすいことの方が重要です。
もし頻繁にスペックファイルをスクロールしていることに気付いたら、それはちょっとぐらいリピートしても問題ないというサインです。
アプリケーションに堅牢なモデルスペックを揃えたので、あなたは順調にコードの信頼性 を上げてきています。
```

## user_spec.rb 記述忘れ

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
  # 追加
  it "is invalid without an email address" do
    user = User.new(email: nil)
    user.valid?
    expect(user.errors[:email]).to include("can't be blank")
  end
  # ここまで

  # 重複したアドレスなら無効な状態であること
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

  # ユーザーのフルネームを文字列として返すこと
  it "returns a user's full name as a string" do
    user = User.new(
      first_name: 'John',
      last_name: "Doe",
      email: "johndoe@example.com",
    )
    expect(user.name).to eq "John Doe"
  end
end
```
