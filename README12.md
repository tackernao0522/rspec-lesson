## shared_context (contextの共有) p144〜

let を使うと複数のテストで必要な共通のテストデータを簡単にセットアップすることができます。<br>
一方、shared_context を使うと複数のテストファイルで必要なセットアップを行うことができます。<br>

タスクコントローラのスペックを見てください。ここで各テストの前に実行されている before ブロックが shared_context に抜き出す候補のひとつになります。<br>
ですがその前に、インスタンス変数のかわりに let を使うようにリファクタリングしておいた方が良さそうです。<br>
というわけで、スペックを次のように変更してください。<br>

+ `spec/controllers/tasks_controller_spec.rb`を編集('@'を削除)<br>

```rb:tasks_controller_spec.rb
require 'rails_helper'

RSpec.describe TasksController, type: :controller do
  # 編集
  let(:user) { FactoryBot.create(:user) }
  let(:project) {FactoryBot.create(:project, owner: user) }
  let(:task) { project.tasks.create!(name: "Test task") }
  # ここまで

  describe "#show" do
    # JSON 形式でレスポンスを返すこと
    it "responds with JSON formatted output" do
      sign_in user
      get :show, format: :json,
        params: { project_id: project.id, id: task.id }
      expect(response.content_type).to include "application/json"
    end
  end

  describe "#create" do
    # JSON 形式でレスポンスを返すこと
    it "responds with JSON formatted output" do
      new_task = { name: "New test task" }
      sign_in user
      post :create, format: :json,
        params: { project_id: project.id, task: new_task }
      expect(response.content_type).to include "application/json"
    end

    # 新しいタスクをプロジェクトに追加すること
    it "adds a new task to the project" do
      new_task = { name: "New test task" }
      sign_in user
      expect {
        post :create, format: :json,
          params: { project_id: project.id, task: new_task }
      }.to change(project.tasks, :count).by(1)
    end

    # 認証を要求すること
    it "requires authentication" do
      new_task = { name: "New test task" }
      # ここではあえてログインしない...
      expect {
        post :create, format: :json,
          params: { project_id: project.id, task: new_task }
      }.to_not change(project.tasks, :count)
      expect(response).to_not be_successful
    end
  end
end
```

+ `$ bundle exec rspec`を実行(パスする)<br>

この最初のリファクタリングで重要な手順は次の通りです。まず before ブロックの中にあった3行をブロックの外に移動します。<br>
それからインスタンス変数を作成するかわりに let を使うように変更します。そして、ファイル内のインスタンス変数を順番に書き換えます。<br>
これはファイル内のインスタンス変数を「検索と置換」すれば OK ですね(たとえば、 @project は project に置換します)。<br>
スペックを実行してテストが引き続きパスすることを確認してください。次に、`spec/support/contexts/project_setup.rb` を新たに作成し、次のような context を書いてください。<br>

+ `mkdir spec/support/contexts && touch $_/project_setup.rb`を実行<br>

+ `spec/support/contexts/project_setup.rb`を編集<br>

```rb:project_setup.rb
RSpec.shared_context "project setup" do
  let(:user) { FactoryBot.create(:user) }
  let(:project) { FactoryBot.create(:project, owner: user) }
  let(:task) { project.tasks.create!(name: "Test task") }
end
```

最後にコントローラスペックに戻り、最初に出てくる3行のletを次のような1行に置き換えてください。<br>

+ `spec/controllers/tasks_controller_spec.rb`を編集<br>

```rb:tasks_controller_spec.rb
require 'rails_helper'

RSpec.describe TasksController, type: :controller do
  include_context "project setup" # 編集

  describe "#show" do
    # JSON 形式でレスポンスを返すこと
    it "responds with JSON formatted output" do
      sign_in user
      get :show, format: :json,
        params: { project_id: project.id, id: task.id }
      expect(response.content_type).to include "application/json"
    end
  end

  describe "#create" do
    # JSON 形式でレスポンスを返すこと
    it "responds with JSON formatted output" do
      new_task = { name: "New test task" }
      sign_in user
      post :create, format: :json,
        params: { project_id: project.id, task: new_task }
      expect(response.content_type).to include "application/json"
    end

    # 新しいタスクをプロジェクトに追加すること
    it "adds a new task to the project" do
      new_task = { name: "New test task" }
      sign_in user
      expect {
        post :create, format: :json,
          params: { project_id: project.id, task: new_task }
      }.to change(project.tasks, :count).by(1)
    end

    # 認証を要求すること
    it "requires authentication" do
      new_task = { name: "New test task" }
      # ここではあえてログインしない...
      expect {
        post :create, format: :json,
          params: { project_id: project.id, task: new_task }
      }.to_not change(project.tasks, :count)
      expect(response).to_not be_successful
    end
  end
end
```

+ `$ bundle exec rspec`を実行(パスする)<br>

スペックを実行するとテストはパスするはずです。ではあえてテストを失敗させてみましょう。<br>
スペックの一つで to を to_not に変えてみてください。予想どおり失敗すると思います。<br>

+ `spec/controllers/tasks_controller_spec.rb`を編集<br>

```rb:tasks_controller_spec.rb
require 'rails_helper'

RSpec.describe TasksController, type: :controller do
  include_context "project setup"

  describe "#show" do
    # JSON 形式でレスポンスを返すこと
    it "responds with JSON formatted output" do
      sign_in user
      get :show, format: :json,
        params: { project_id: project.id, id: task.id }
      expect(response.content_type).to include "application/json"
    end
  end

  describe "#create" do
    # JSON 形式でレスポンスを返すこと
    it "responds with JSON formatted output" do
      new_task = { name: "New test task" }
      sign_in user
      post :create, format: :json,
        params: { project_id: project.id, task: new_task }
      expect(response.content_type).to_not include "application/json" # to_notに変えてみる 確認後戻す
    end

    # 新しいタスクをプロジェクトに追加すること
    it "adds a new task to the project" do
      new_task = { name: "New test task" }
      sign_in user
      expect {
        post :create, format: :json,
          params: { project_id: project.id, task: new_task }
      }.to change(project.tasks, :count).by(1)
    end

    # 認証を要求すること
    it "requires authentication" do
      new_task = { name: "New test task" }
      # ここではあえてログインしない...
      expect {
        post :create, format: :json,
          params: { project_id: project.id, task: new_task }
      }.to_not change(project.tasks, :count)
      expect(response).to_not be_successful
    end
  end
end
```

+ `$ bundle exec rspec`を実行<br>

```:terminal
Failures:

  1) TasksController#create responds with JSON formatted output
     Failure/Error: expect(response.content_type).to_not include "application/json"
       expected "application/json; charset=utf-8" not to include "application/json"
     # ./spec/controllers/tasks_controller_spec.rb:23:in `block (3 levels) in <main>'

Finished in 5.49 seconds (files took 2.4 seconds to load)
48 examples, 1 failure

Failed examples:

rspec ./spec/controllers/tasks_controller_spec.rb:18 # TasksController#create responds with JSON formatted output
```

これはあまり読みやすいとは言えませんね。この点については後ほど対処します。<br>
とりあえず、次は別の方法でテストを失敗させてみましょう。to_not を to に戻し、別の Content-Type を与えてみます。<br>
たとえば、:html や :csv などです。こちらもやはり失敗します。そしてエラーメッセージもあまり読みやすくありません。<br>

```:terminal
1) TasksController#show responds with JSON formatted output Failure/Error: expect(response).to have_content_type :csv
expected #<ActionDispatch::TestResponse:0x007fc6d1d353c0 @mon_owner=nil, @mon_count=0, @mon_mutex=#<Thread::Mu...:Headers:0x007fc6d1d0f170 @req=#<ActionController::TestRequest:0x007fc6d1d356b8 ...>>, @variant=[]>> to have content type :csv
```

では今から読みやすさを改善していきましょう。まず最初に Content-Type のハッシュを match メソッドの外に切り出します。<br>
RSpec ではマッチャー内でヘルパーメソッドを定義し、コードをきれいにすることができます。<br>

+ `mkdir spec/support/matchers && touch $_/content_type.rb`を実行<br>

+ `spec/support/matchers/content_type.rb`を編集<br>

```rb:content_type.rb
RSpec::Matchers.define :have_content_type do |expected|
  match do |actual|
    begin
      actual.content_type.include? content_type(expected)
    rescue => ArgumentError
      false
    end
  end

  def content_type(type)
    types = {
      html: "text/html",
      json: "application/json",
    }
    types[type.to_sym] || "unknown content type"
  end
end
```

+ `$ bundle exec rspec`を実行(失敗するテストを実行<br>

もう一度テストを実行し、先ほどと同じ方法でテストを失敗させてみてください。<br>
マッチャのコードはちょっと読みやすくなりましたが、出力はまだ読みやすくありません(訳 注: 同じ出力結果になります)。この点も改善可能です。<br>
RSpec のカスタムマッチャの DSL では match メソッドに加えて、失敗メッセージ(failure message )と、否定の失敗メッセージ(negated failure message )を定義するメソッドが用意されています。<br>
つまり、to や to_not で失敗したときの報告方法を定義できるのです。<br>

+ `spec/support/matchers/content_type.rb`を編集 p150〜<br>

```rb:content_type.rb
RSpec::Matchers.define :have_content_type do |expected|
  match do |actual|
    begin
      actual.content_type.include? content_type(expected)
    rescue => ArgumentError
      false
    end
  end

  # 追加
  failure_message do |actual|
    "Expected \"#{content_type(actual.content_type)} " +
    "(#{actual.content_type})\" to be Content Type " +
    "\"#{content_type(expected)}\" (#{expected})"
  end

  failure_message_when_nageted do |actural|
    "Expected \"#{content_type(actual.content_type)} " +
    "(#{actual.content_type})\" to not be Content Type " +
    "\"#{content_type(expected)}\" (#{expected})"
  end
  # ここまで

  def content_type(type)
    types = {
      html: "text/html",
      json: "application/json",
    }
    types[type.to_sym] || "unknown content type"
  end
end
```

テストは引き続きパスするはずです。ですが、わざと失敗させてみると、出力内容が改善されています。<br>

+ `$ bundle exec rspec spec/controllers`を実行(失敗するテストを実行)<br>

```:terminal
Failures:
1) TasksController#show responds with JSON formatted output Failure/Error: expect(response).to_not have_content_type :json
Expected "unknown content type (application/json; charset=utf-8)" to not be Content Type "application/json" (json)
```

ちょっと良くなりましたね。結果として受け取ったレスポンス(application/json の Content-Type )はマッチャに渡した Content-Type を含んでいます。<br>
ですが、(わざと失敗させているため)それは期待していないレスポンスです。スペックに戻って to_not を to に戻し、それ から :json のかわりに :html を渡してください。スペックを実行してみましょう。<br>

+ `spec/controllers/tasks_controller_spec.rb`を編集<br>

```rb:tasks_controller_spec.rb
require 'rails_helper'

RSpec.describe TasksController, type: :controller do
  include_context "project setup"

  describe "#show" do
    # JSON 形式でレスポンスを返すこと
    it "responds with JSON formatted output" do
      sign_in user
      get :show, format: :json,
        params: { project_id: project.id, id: task.id }
      expect(response.content_type).to include "application/json"
    end
  end

  describe "#create" do
    # JSON 形式でレスポンスを返すこと
    it "responds with JSON formatted output" do
      new_task = { name: "New test task" }
      sign_in user
      post :create, format: :html, # htmlにしてみる
        params: { project_id: project.id, task: new_task }
      expect(response.content_type).to include "application/json"
    end

    # 新しいタスクをプロジェクトに追加すること
    it "adds a new task to the project" do
      new_task = { name: "New test task" }
      sign_in user
      expect {
        post :create, format: :json,
          params: { project_id: project.id, task: new_task }
      }.to change(project.tasks, :count).by(1)
    end

    # 認証を要求すること
    it "requires authentication" do
      new_task = { name: "New test task" }
      # ここではあえてログインしない...
      expect {
        post :create, format: :json,
          params: { project_id: project.id, task: new_task }
      }.to_not change(project.tasks, :count)
      expect(response).to_not be_successful
    end
  end
end
```

+ `$ bundle exec rspec spec/controllers`を実行<br>

```:terminal
Failures:
1) TasksController#show responds with JSON formatted output Failure/Error: expect(response).to have_content_type :html
Expected "unknown content type (application/json; charset=utf-8)" to be Content Type "text/html" (html)
```

すばらしい!読みやすさが改善されました。最後にもうひとつだけ改善しておきましょう。<br>
have_content_type はちゃんと動作していますが、be_content_type でも動くようにすると良いかもしれません。<br>
マッチャはエイリアスを作ることができます。カスタムマッチャの 最終バージョンはこのようになります。<br>

+ `spec/support/matchers/content_type.rb`を編集<br>

```rb:content_type.rb
RSpec::Matchers.define :have_content_type do |expected|
  match do |actual|
    begin
      actual.content_type.include? content_type(expected)
    rescue => ArgumentError
      false
    end
  end

  failure_message do |actual|
    "Expected \"#{content_type(actual.content_type)} " +
    "(#{actual.content_type})\" to be Content Type " +
    "\"#{content_type(expected)}\" (#{expected})"
  end

  failure_message_when_nageted do |actural|
    "Expected \"#{content_type(actual.content_type)} " +
    "(#{actual.content_type})\" to not be Content Type " +
    "\"#{content_type(expected)}\" (#{expected})"
  end

  def content_type(type)
    types = {
      html: "text/html",
      json: "application/json",
    }
    types[type.to_sym] || "unknown content type"
  end
end

RSpec::Matchers.alias_matcher :be_content_type, :have_content_type # 追加
```

以上でカスタムマッチャは完成です。これはいいアイデアでしょうか?たしかにテストは確実に読みやすくなりました。<br>
「レスポンスが Content-Type JSON になっていることを期待する(Expect response to be content type JSON )」は、「レスポンスの Content-Type が application/json を含んでいる(Expect response content type to include application/json )」よりも改善されています。<br>
ですが、新しいマッチャがあると、メンテナンスしなければならないコードが増えます。<br>
カスタムマッチャにはその価値があるでしょうか? その結論はあなたとあなたのチームで決める必要があります。<br>
しかし何にせよ、これでみなさんは必要になったときにカスタムマッチャを作ることができるようになりました。<br>

`＊`<br>
カスタムマッチャ作りにハマっていく前に、shoulda-matchers gem も一度見ておいてください。<br>
この gem はテストをきれいにする便利なマッチャをたくさん提供してくれます。<br>
特に役立つのがモデルとコントローラのスペックです。みなさんが必要とするマッチャは、すでにこの gem で提供されているかもしれません。<br>
たとえば、第3章で書いたスペックのいくつかは、it { is_expected.to validate_presence_of :name } のように、短くシンプルに書くことができます。<br>

## aggregate_failures (失敗の集約) p 153〜

この章よりも前の章で、私はモデルとコントローラのスペックは各 example につきエクスペクテーションを一つに制限した方が良いと書きました。<br>
一方、システムスペックとリクエストスペックでは、機能の統合がうまくできていることを確認するために、必要に応じてエクスペクテーションをたくさん書いても良い、と書きました。<br>
ですが、単体テストでもいったんコーディングが完了してしまえば、この制限が必ずしも必要ないことがあります。<br>
また、 統合テストでも Launchy(第6章 参照)に頼ることなく、失敗したテストのコンテキストを収集すると役に立つことが多いです。<br>
ここで問題となるのは、RSpec はテスト内で失敗するエクスペクテーションに遭遇すると、そこで即座に停止して失敗を報告することです。<br>
残りのステップは実行されません。しかし、RSpec 3.3では aggregate_failures (失敗の集約)という機能が導入され、他のエクスペクテーションも続けて実行することができます。<br>
これにより、そのエクスペクテーションが失敗し た原因がさらによくわかるかもしれません。<br>
まず、aggregate_failures によって、低レベルのテストがきれいになるケースを見てみましょう。<br>
第5章では Project コントローラを検証するためにこのようなテストを書きました。<br>

+ `spec/controllers/projects_controller_spec.rb`(既存確認)<br>

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

このふたつのテストでやっていることはほとんど同じです(第5章でもそのように説明し ています)。なので、どちらか一方を選択して、二つのテストを一つに集約することができます。<br>
まずは説明のために、二つのテストを次のようにまとめてください。その際、sign_in のステップをコメントアウトすることをお忘れなく(これは一時的な変更です)。<br>

+ `spec/controllers/projects_controller_spec.rb`を編集 p154〜<br>

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
        # sign_in @user // コメントアウト
        get :index
        expect(response).to be_successful
        expect(response).to have_http_status "200" # 追加
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

+ `$ bundle exec rspec spec/controllers`を実行(失敗する)<br>

このスペックを実行すると予想通り最初のエクスペクテーションで失敗します。<br>
二番目の エクスペクテーションも失敗するはずですが、このままでは絶対に実行されません。<br>
そこで、 この二つのエクスペクテーションを集約してみましょう。<br>

+ `spec/controllers/projects_controller_spec.rb`を編集 p154〜<br>

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
        # sign_in @user 失敗を確認したら戻す
        get :index
        aggregate_failures do # 追加
          expect(response).to be_successful
          expect(response).to have_http_status "200"
        end # 追加
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

+ `$ bundle exec rspec spec/controllers`を実行(失敗する)<br>

```:terminal
Failures:

  1) ProjectsController#index as an authenticated user response successfully
     Got 2 failures from failure aggregation block.
     # ./spec/controllers/projects_controller_spec.rb:15:in `block (4 levels) in <main>'

     1.1) Failure/Error: expect(response).to be_successful
            expected `#<ActionDispatch::TestResponse:0x000000010b502880 @mon_data=#<Monitor:0x000000010b5027e0>, @mon_data_...e, @cache_control={}, @request=#<ActionController::TestRequest GET "http://test.host/" for 0.0.0.0>>.successful?` to be truthy, got false
          # ./spec/controllers/projects_controller_spec.rb:16:in `block (5 levels) in <main>'

     1.2) Failure/Error: expect(response).to have_http_status "200"
            expected the response to have status code 200 but it was 302
          # ./spec/controllers/projects_controller_spec.rb:17:in `block (5 levels) in <main>'

Finished in 1.1 seconds (files took 2.23 seconds to load)
21 examples, 1 failure
```

すばらしい、これでどちらのエクスペクテーションも実行されました。<br>
そして、どうしてレスポンスが成功として返ってこなかったのか、追加の情報を得ることもできました。<br>
sign_in の行のコメントを外し、テストをグリーンに戻してください。<br>
私は統合テストで aggregate_failures をよく使います。<br>
aggregate_failures を使えば、同じコードを何度も実行して遅くなったり、複雑なセットアップを複数のテストで共有したりせずに、 テストが失敗した複数のポイントを把握することができます。<br>
たとえば第6章 で説明した、プロジェクトを作成するシナリオでエクスペクテーションの一部を集約してみましょう。<br>

+ `spec/system/projects_spec.rb`を編集 p156〜<br>

```rb:projects_spec.rb
require 'rails_helper'

RSpec.describe "Projects", type: :system do
  # include LoginSupport 削除

  # ユーザーは新しいプロジェクトを作成する
  scenario "user creates a new project" do
    user = FactoryBot.create(:user)
    # この章で独自に定義したログインヘルパーを使う場合
    # sign_in user
    # もしくは Deviseが提供しているヘルバーを使う場合
    sign_in user # 追加
    visit root_path

    expect {
      click_link "New Project"
      fill_in "Name", with: "Test Project"
      fill_in "Description", with: "Trying out Capybara"
      click_button "Create Project"
    }.to change(user.projects, :count).by(1)

    aggregate_failures do # 追加
      expect(page).to have_content "Project was successfully created"
      expect(page).to have_content "Test Project"
      expect(page).to have_content "Owner: #{user.name}"
    end # 追加
  end
end
```

こうすると、何らかの原因でフラッシュメッセージが壊れても、残りの二つのエクスペクテーションは続けて実行されます。<br>
ただし、注意すべき点がひとつあります。<br>
それは、aggregate_failures は失敗するエクスペクテーションにだけ有効に働くのであって、テストを実行するために必要な一般的な実行条件には働かないと言うことです。<br>
上の例で言うと、もし何かがおかしくなって New Project のリンクが正しくレンダリングされなかった場合は、Capybara はエラーを報告します。<br>

`例`<br>

```:terminal
Failures:
1) Projects user creates a new project Failure/Error: click_link "New Project"
     Capybara::ElementNotFound:
       Unable to find link "New Project"
```

言い換えるなら、ここでやっているのはまさに失敗するエクスペクテーションの集約であり、失敗全般を集約しているわけではありません。<br>
とはいえ、私は aggregate_failures を気 に入っており、自分が書くテストではこの機能をよく使っています。<br>

## テストの可読性を改善する

統合テストはさまざまな構成要素を検証します。UIも(JavaScriptとの実行ですらも)、アプリケーションロジックも、データベース操作も、ときには外部システムとの連携も、全部ひとつのテストで検証できます。<br>
これはときに、だらだらしたテストコードになったり、テストコードが読みづらくなったり、何が起きているか理解するために手前や後ろのコードを行ったり来たりすることにつながります。<br>
第6章で作成したテストを見直してください。このテストではタスクの完了を実行するUIを検証しました。<br>
その中でも特に、タスクの完了や未完了をマークするステップと、その処理が期待どおりに実行されたか検証するやり方を見直してみてください。<br>

+ `spec/system/tasks_spec.rb`を再掲 p 158〜<br>

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

これぐらいであればそこまで可読性は悪くないかもしれません。ですが、この機能が今後もっと複雑になったらどうでしょうか?<br>
たとえば、タスクが完了したときにもっと詳細な情報を記録する必要が出てきた場合を考えてみてください。<br>
タスクを完了状態にしたユーザーの情報や、いつタスクが完了したのか、といった情報を記録する必要が出てきた場合などです。<br>
この場合、新しく追加された属性に対してそれぞれ、データベースの状態と UI の表示を確認するために新しい行を追加する必要が出てきます。<br>
また、チェックボックスのチェックが外されたときは逆の検証を同じようにする必要があります。<br>
こうなると、今はまだ小さいテストも、あっという間に⻑くなってしまいます。<br>
このような場合は、スペックを読むときにコードの前後を行ったり来たりせずに済むよう、各ステップを独立したヘルパーメソッドに抽出することができます。<br>
新しくエクスペクテーションを追加する場合は、テストコード内に直接埋め込むのではなく、このヘルパーメソッドに追加します。<br>
これは「シングルレベルの抽象化を施したテスト([testing at a single level of abstraction](https://thoughtbot.com/blog/acceptance-tests-at-a-single-level-of-abstraction) )」として知られるテクニックです。<br>
このテクニックの基本的な考えはテストコード全体を、内部で何が起きているのか抽象的に理解できる名前を持つメソッドに分割することです。<br>
あくまで抽象化が目的なので、内部の詳細を見せる必要はありません。<br>
では、spec/system/tasks_spec.rb 内のだらだらしたコードを次のように整理してみましょう。<br>

+ `spec/system/tasks_spec.rb`を編集 p159〜`を編集<br>

```rb:tasks_spec.rb
require 'rails_helper'

RSpec.describe "Tasks", type: :system do
  let(:user) { FactoryBot.create(:user) }
  let(:project) {
    FactoryBot.create(:project,
      name: "RSpec tutorial",
      owner: user
    )
  }
  let!(:task) { project.tasks.create!(name: "Finish RSpec tutorial") }

  # ユーザーがタスクの状態を切り替える
  scenario 'user toggles a task', js: true do
    sign_in user
    go_to_project "RSpec tutorial"

    complete_task "Finish RSpec tutorial"
    expect_complete_task "Finish RSpec tutorial"

    undo_complete_task "Finish RSpec tutorial"
    expect_incomplete_task "Finish RSpec tutorial"
  end

  def go_to_project(name)
    visit root_path
    click_link name
  end

  def complete_task(name)
    check name
  end

  def undo_complete_task(name)
    uncheck name
  end

  def expect_complete_task(name)
    aggregate_failures do
      expect(page).to have_css "label.completed", text: name
      expect(task.reload).to be_completed
    end
  end

  def expect_incomplete_task(name)
    aggregate_failures do
      expect(page).to_not have_css "label.completed", text: name
      expect(task.reload).to_not be_completed
    end
  end
end
```

+ `$ bundle exec rspec`を実行(パスする)<br>

テストをもっと読みやすくするために、この新バージョンでは何カ所かリファクタリングをしています。<br>
まず、テストデータの作成はテスト内から上側の let と let! メソッドに移動させています。<br>
それからテストの各ステップを別々のメソッドに切り出しました。ユーザーとしてログインし、続いてテスト対象のプロジェクトに移動します。<br>
それから最後に目的のステップを実行し、各ステップについて独自に作成したエクスペクテーションを実行します。<br>
具体的にはタスクを完了済みにし、本当に完了済みになっているか確認します。そして 完了済みのタスクを元に戻し、今度は未完了になっているか確認します。<br>
テストはとてもシンプルに読めるようになりました。私たちが実際にどうやって必要なアクションを実行しているのかという詳細は、別のメソッドに追い出されています。<br>
どうでしょう、よくなりましたか?個人的には新しいテストはとても読みやすくなっている点が気に入っています。<br>
もしかするとプログラミングを知らない人がこのテストを読んでも、何をやっているのかある程度理解できるかもしれません。<br>
もしタスクを完了したり未完了にしたりする方法を変更する場合でも、変更を加えるのはヘルパーメソッドだけで済みます。<br>
これも新バージョンの良いところです。新しく作ったヘルパーメソッドは同じファイルの他のテストで再利用できる点もいいですね。<br>
たとえば、もしタスクを追加したり削除した りするシナリオを追加する場合でも、このヘルパーメソッドを再利用できます(この新しいテストは演習問題としてみなさんにやってもらいます!)。<br>
ですが、このアプローチについてはちょっと不満に感じている点もあります。私はこの変 更でテストデータを準備するためにlet! を使ったところがあまり好きではありません。<br>
これは必ずしも「シングルレベルの抽象化を施したテスト」の良い効果とは言えないでしょう。<br>
この点については、ファイルにさらにシナリオを追加していく際に、テストデータの作成方法を引き続き改善していけると思います。<br>
また、私は過去にこのアプローチが過剰に使われているケースを見てきました。<br>
過剰に使われてしまうと、ヘルパーメソッドが何をやって、何を検証しているのか理解するのにテストスイートの中身を詳細に確認しなければなりません。<br>
私に言わせれば、こうなってしまうと可読性を上げるための努力が、かえって可読性を下げることになっています。<br>
私は「シングルレベルの抽象化を施したテスト」の考え方は好きです。ですが、みなさんは RSpec や Capybara に慣れるまで、必ずしも使う必要はないと考えています。<br>
ここで覚えておいてほしいことは、アプリケーション側のコードと同様、テストスイートのコードも自分のスキルが向上するにつれて改善されていく、ということです。<br>

## まとめ

この章ではテスト内、もしくは複数のテストファイルにまたがるコードの重複を減らすアプローチをいくつか見てきました。<br>
もうお気づきかもしれませんが、どのアプローチも効果的なテストスイートを構築するために必須というわけではありません。<br>ですが、こうしたアプローチを理解し、賢く適用していけば、⻑い目で見たときに保守しやすいテストスイートになります。<br>
これは Rails のコードベースに Rails のベストプラクティスを適用しておけば、ずっと快適に開発できるのと同じです。<br>

## 演習問題

+ 今度は現状のプロジェクト用のシステムスペックを見てください。<br>
  もしプロジェクトを編集する場合のテストを追加するとしたら、既存の「ユーザーは新しいプロジェクトを作成する(user creates a new project )」シナリオからどのステップを再利用しますか?<br>
  みなさんはこのシナリオ用のテストを追加できますか?<br>
  ただし、その際はこの章で紹介した、テストを DRY にするテクニックも一緒に使ってください。<br>
  もちろん、可読性と保守性は維持する必要があります。<br>
