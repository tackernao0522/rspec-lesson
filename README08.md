## ユーザー入力のエラーをテストする p97〜

```
ここまでに追加した認可済みのユーザーに対するテストを思い出してください。
ここまで私たちは正常系の入力しかテストしませんでした。ユーザーはプロジェクトを作成、または 編集するために有効な属性値を送信したので、Rails は正常にレコードを作成、または更新できました。
ですが、モデルスペックの場合と同じように、何か正しくないことがコントローラ内で起こったときも意図した通りの動きになるか検証するのは良い考えす。
今回の場合だと、もしプロジェクトの作成、または編集時にバリデーションエラーが発生したら、何が起きるでしょうか?
こういうケースの一例として、認可済みのユーザーが create アクションにアクセスしたときのテストを少し変更してみましょう。
まず、テストを二つの新しい context に分割することから始めます。
一つは有効な属性値で、もう一つは無効な属性値です。既存のテストは最初の context に移動し、無効な属性については新しくテストを追加します。
```

+ `spec/controllers/projects_controller_spec.rb`を編集<br>

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

      # プロジェクトを追加できること
      it "adds a project" do
        project_params = FactoryBot.attributes_for(:project)
        sign_in @user
        expect {
          post :create, params: { project: project_params }
        }.to change(@user.projects, :count).by(1)
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
        project_params = FactoryBot.attributes_for(:project)
        post :create, params: { project: project_params }
        expect(response).to redirect_to "/users/sign_in"
      end
    end
  end

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

  # 追加
  describe "#create" do
    # 認可済みのユーザーとして
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
            post :create, params: { project: project_params }
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
            post :create, params: { project: project_params }
          }.to_not change(@user.projects, :count)
        end
      end
    end

  end
  # ここまで

  describe "#update" do
    # 認可されたユーザーとして
    context "as an authorized user" do
      before do
        @user = FactoryBot.create(:user)
        @project = FactoryBot.create(:project, owner: @user)
      end

      # プロジェクトを更新できること
      it "updates a project" do
        project_params = FactoryBot.attributes_for(:project, name: "New Project Name")
        sign_in @user
        patch :update, params: { id: @project.id, project: project_params }
        expect(@project.reload.name).to eq "New Project Name"
      end
    end

    # 認可されていないユーザーとして
    context "as an unauthorized user" do
      before do
        @user = FactoryBot.create(:user)
        other_user = FactoryBot.create(:user)
        @project = FactoryBot.create(:project, owner: other_user, name: "Same Old Name")
      end

      # プロジェクトを更新できないこと
      it "does not update the project" do
        project_params = FactoryBot.attributes_for(:project, name: "New Name")
        sign_in @user
        patch :update, params: { id: @project.id, project: project_params }
        expect(@project.reload.name).to eq "Same Old Name"
      end

      # ダッシュボードへリダイレクトすること
      it "redirects to the dashboard" do
        project_params = FactoryBot.attributes_for(:project)
        sign_in @user
        patch :update, params: { id: @project.id, project: project_params }
        expect(response).to redirect_to root_path
      end
    end

    # ゲストとして
    context "as a guest" do
      before do
        @project = FactoryBot.create(:project)
      end

      # 302レスポンスを返すこと
      it "returns a 302 response" do
        project_params = FactoryBot.attributes_for(:project)
        patch :update, params: { id: @project.id, project: project_params }
        expect(response).to have_http_status "302"
      end

      # サインイン画面にリダイレクトすること
      it "redirects to the sign-in page" do
        delete :destroy, params: { id: @project.id }
        expect(response).to redirect_to "/users/sign_in"
      end

      # プロジェクトを削除できないこと
      it "does not delete the project" do
        expect {
          delete :destroy, params: { id: @project.id }
        }.to_not change(Project, :count)
      end
    end
  end
end
```

```
新しいテストではプロジェクトファクトリの新しいトレイトも使っています。
こちらも追加しておきましょう。
```

+ `spec/factories/projects.rb`を編集<br>

```rb:projects.rb
FactoryBot.define do
  factory :project do
    sequence(:name) { |n| "Test Project #{n}" }
    description { "Sample project for testing purposes" }
    due_on { 1.week.from_now }
    association :owner

    # メモ付きのプロジェクト
    trait :with_notes do
      after(:create) { |project| create_list(:note, 5, project: project) }
    end

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

    # 追加
    # 無効になっている
    trait :invalid do
      name { nil }
    end
    # ここまで
  end
end
```

+ `$ bundle exec rspec spec/controllers`を実行(パスする)<br>

```
これで create アクションを実行したときに、名前のないプロジェクトの属性値が送信さ
れます。
この場合、コントローラは新しいプロジェクトを保存しません。
```

## HTML以外の出力を扱う p99〜

```
コントローラの責務はできるだけ小さくすべきです。ただし、コントローラが担うべき責務の一つに、適切なフォーマットでデータを返す、という役割があります。ここまでにテストしたコントローラのアクションはすべて text/html フォーマットでデータを返していました。
ですが、テストの中ではそのことを特に意識していませんでした。
簡単に説明するために、ここでは Task コントローラを見ていきます。
Task コントローラは Rails の scaffold で作成され、デフォルトで定義された CRUD アクションにはほとんど変更を加えていません。
ですので、HTML と JSON の両方のフォーマットでリクエストを受け付け、レスポンスを返すことができます。
JSON に限定したテストを書くとどうなるか、今から見ていきましょう。
Task コントローラ用のスペックファイルはまだ作成していません。
ですが、ジェネレータを使えば簡単に作成できます(bin/rails g rspec:controller tasks --controller-specs --no-request-specs )。
それから、ここまでに学んだ認証機能のテストと、データを送信するテストの知識を使えば、シンプルなテストを追加することができます。
今回のテストは JSON を扱うコントローラのテストを網羅的に説明するものではありません。
ですが、コントローラのスペックファイルで何をどうすればいいか、という参考情報にはなると思います。
まず、コントローラの show アクションを見てみましょう。
```

+ `$ bin/rails g rspec:controller tasks --controller-specs --no-request-specs`を実行<br>

+ `spec/controllers/tasks_controller_spec.rb`を編集<br>

```rb:tasks_controller_spec.rb
require 'rails_helper'

RSpec.describe TasksController, type: :controller do
  before do
    @user = FactoryBot.create(:user)
    @project = FactoryBot.create(:project, owner: @user)
    @task = @project.tasks.create!(name: "Test task")
  end

  describe "#show" do
    # JSON 形式でレスポンスを返すこと
    it "responds with JSON formatted output" do
      sign_in @user
      get :show, format: :json,
        params: { project_id: @project.id, id: @task.id }
      expect(response.content_type).to include "application/json"
    end
  end

end
```

+ `$ bundle exec rspec spec/controllers/tasks_controller_spec.rb`を実行(パスする)<br>

```
セットアップはこの章ですでに説明した他のスペックとほとんど同じです。
必要なデータはユーザーとプロジェクト(ユーザーがアサインされる)とタスク(プロジェクトがアサ インされる)の3つです。
それから、テストの中でユーザーをログインさせ、GET リクエストを送信してコントローラの show アクションを呼びだしています。
このテストのちょっとだけ新しい点は、デフォルトの HTML 形式のかわりに format: :json というオプションで JSON 形式であることを指定しているところです。
こうするとコントローラは言われたとおりにリクエストを処理します。つまり、application/json の Content-Type でレスポンスを返してくれるのです。
ただし、厳密には application/json; charset=utf-8 のように文字コード情報も一緒に付いてきます。
そこで、このテストでは include マッチャを使ってレスポンスの中に application/json が含まれていればテストがパスするようにしました。
意図した通りにテストできていることを確認するため、application/json を text/html に変えてみましょう。案の定、テストは失敗するはずです。
次に、create アクションが JSON を処理できることを確認するテストをいくつか追加してみましょう。
```

p100〜<br>

+ `spec/controllers/tasks_controller_spec.rb`を編集<br>

```rb:tasks_controller_spec.rb
require 'rails_helper'

RSpec.describe TasksController, type: :controller do
  before do
    @user = FactoryBot.create(:user)
    @project = FactoryBot.create(:project, owner: @user)
    @task = @project.tasks.create!(name: "Test task")
  end

  describe "#show" do
    # JSON 形式でレスポンスを返すこと
    it "responds with JSON formatted output" do
      sign_in @user
      get :show, format: :json,
        params: { project_id: @project.id, id: @task.id }
      expect(response.content_type).to include "application/json"
    end
  end

  # 追加
  describe "#create" do
    # JSON 形式でレスポンスを返すこと
    it "responds with JSON formatted output" do
      new_task = { name: "New test task" }
      sign_in @user
      post :create, format: :json,
        params: { project_id: @project.id, task: new_task }
      expect(response.content_type).to include "application/json"
    end

    # 新しいタスクをプロジェクトに追加すること
    it "adds a new task to the project" do
      new_task = { name: "New test task" }
      sign_in @user
      expect {
        post :create, format: :json,
          params: { project_id: @project.id, task: new_task }
      }.to change(@project.tasks, :count).by(1)
    end

    # 認証を要求すること
    it "requires authentication" do
      new_task = { name: "New test task" }
      # ここではあえてログインしない...
      expect {
        post :create, format: :json,
          params: { project_id: @project.id, task: new_task }
      }.to_not change(@project.tasks, :count)
      expect(response).to_not be_successful
    end
  end
  # ここまで
end
```

+ `$ bundle exec rspec spec/controllers/tasks_controller_spec.rb`を実行(パスする)<br>

```
セットアップは同じですが、今回は POST リクエストをコントローラの create アクションに送信しています。
また、この章ですでに説明した方法で task のパラメータも一緒に送信しています。
そして、ここでもやはり JSON 形式でリクエストを送信するようにオプションを指定する必要があります。
それから、JSON 形式でも「リクエストを送信して本当にデータベースに保存されるか?」もしくは「ユーザーがログインしていない状態であればデータベースへの保存が中断されるか?」というようなチェックができます。
```

## まとめ

```

この章ではアプリケーションに数多くのテストを追加しました。
テストしたコントローラはたった二つだけなんですけどね!コントローラのテストは簡単に追加していくことができます。
ですが、すぐに大きくなって手に負えなくなることもよくあります。
今回は実際のアプリケーションでよくあるコントローラのテストシナリオをいくつか説明しました。
ですが、私が開発しているアプリケーションでは、コントローラのテストはアクセス制御が正しく機能しているか確認するテストに限定するようにしています。
この章で説明したテストでいうと、認可されていないユーザーとゲストに対するテストが該当します。
認可されているユーザーに関しては、より上のレベルのテストで検証できます(この内容は 次の章で説明します)。
また、コントローラのテストは Rails や RSpec から完全にはなくなっていないものの、最近では時代遅れのテストになってしまいました。
Project コントローラの destroy アクションを テストしたときのことを思い出してください。
あれはテストは作ったものの、結局 UI が用 意されていなかった、というオチでした。この件はコントローラのテストに限界があることの一例です。
私からのアドバイスをまとめると、コントローラのテストは対象となる機能の単体テスト として最も有効活用できるときだけ使うのがよい、ということです。
ただし、使いすぎない ように注意してください。
```

## Q&A

```
successful と http status は両方チェックしないといけませんか?
必ずしも必須ではありません。
どちらかひとつで十分な場合もありますが、それはあなたのコントローラが HTTP クライアントに返すレスポンスの複雑さによります。
```

## 演習問題

```
• 今回はProjectコントローラのnewやeditはテストしませんでした。このテストを追加 してみてください。ヒント:このテストは show アクションによく似たものになります。

• あなたのアプリケーションのコントローラについて、どのメソッドがどのユーザーに対してアクセスを許可するか、表にまとめてください。
たとえば、私が有料コンテンツを 扱うブログアプリケーションを作っていたとします。ブログ記事にアクセスするためには、ユーザーは会員にならなければいけません。
ですが、その記事を読みたくなるようにタイトルの一覧だけは見せて良いことにします。
実際のユーザーは自分に割り当てら れたロールによってアクセスレベルが異なります。
このような架空の Post コントロー ラは次のような権限制御機能になるかもしれません。
```

|役割|Index|Show|Create|Update|Destroy|
|:---:|:---:||:---:|:---:|:---:|:---:|
|管理者|あり|あり|あり|あり|あり|
|編集者|あり|あり|あり|あり|あり|
|筆者|あり|あり|あり|あり|なし|
|会員|あり|あり|なし|なし|なし|
|ゲスト|あり|なし|なし|なし|なし|

```
この一覧表を使って必要なテストシナリオを検討してください。
今回は new と create は一つのカラムにまとめました(なぜなら、何も作成できないのに new で画面を表示しても意味がないからです)。
edit と update も同様です。ただし、index と show は別々にしています。
この表をあなたが開発しているアプリケーションの認証/認可の要件と比較するとどうでしょうか?
どこを変える必要がありますか?
```
