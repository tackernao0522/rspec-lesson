# 11. テスト駆動開発に向けて P201〜

ふう。私たちはプロジェクト管理アプリケーションでたくさんのことをやってきました。 本書の冒頭から欲しかった機能はすでにできあがっていましたが、テストは全くありませんでした。<br>
今ではアプリケーションは十分にテストされていますし、もし穴が残っていたとしても、私たちはその穴を調べて塞ぐだけのスキルを身につけています。<br>
しかし、私たちがやってきたことはテスト駆動開発(TDD)でしょうか?<br>
厳密に言えば「いいえ」です。アプリケーションコードは私たちが最初のスペックを追加する前から存在していました。私たちが今までやって来たことは探索的テストに近いものです。<br>
つまり、アプリケーションをより深く理解するためにテストを使っていました。本当に TDD を練習するにはアプローチを変える必要があります。<br>
すなわち、テストを先に書き、それからテストをパスさせるコードを書き、そしてコードをリファクタリングして今後もずっとコードを堅牢にしていくのです。<br>
その過程で、テストを使ってどういうコーディングをすべきか検討します。私たちはバグのないソフトウェアを作り上げるために努力しています。 <br>
そのソフトウェアは将来新しい要件が発生して変更が入るかもしれません。<br>
さあ、このサンプルアプリケーションで TDD を実践してみましょう!<br>

## フィーチャを定義する

現時点ではプロジェクトを完了済みにする方法は用意されていません。この機能があれば、ユーザーは完了したプロジェクトを見えない場所に片付けて、現在のプロジェクトだけにフォーカスできるようになるので便利そうです。<br>
次のような二つの機能を追加して、この要件を実装してみましょう。<br>

  + プロジェクトを 完了済み にするボタンを追加する。<br>

  + ログイン直後に表示されるダッシュボード画面では完了済みのプロジェクトを表示しないようにする。<br>

まず最初のシナリオから始めましょう。コーディングする前に、テストスイート全体を実行し、機能を追加する前に全部グリーンになることを確認してください。<br>
もしパスしないスペックがあれば、本書で身につけたスキルを活かしてスペックを修正してください。大事なことは開発を進める前にまず、きれいな状態から始められるようにしておくことです。<br>

次に新しいシステムスペックに作業のアウトラインを記述します。プロジェクトを管理するためのシステムテストファイルはすでに作ってあるので、そこに新しいシナリオを追加できます。(状況によっては新しくファイルを作った方がいい場合もあります。)<br>
既存のシステムテストファイルを開き、新しいシナリオのスタブを追加してください。<br>

+ `spec/system/projects_spec.rb`を編集<br>

```rb:projects_spec.rb
require 'rails_helper'

RSpec.describe "Projects", type: :system do
  # include LoginSupport

  # ユーザーは新しいプロジェクトを作成する
  scenario "user creates a new project" do
    user = FactoryBot.create(:user)
    # この章で独自に定義したログインヘルパーを使う場合
    # sign_in user
    # もしくは Deviseが提供しているヘルバーを使う場合
    sign_in user
    visit root_path

    expect {
      click_link "New Project"
      fill_in "Name", with: "Test Project"
      fill_in "Description", with: "Trying out Capybara"
      click_button "Create Project"
    }.to change(user.projects, :count).by(1)

    aggregate_failures do
      expect(page).to have_content "Project was successfully created"
      expect(page).to have_content "Test Project"
      expect(page).to have_content "Owner: #{user.name}"
    end
  end

  # 追加
  # ユーザーはプロジェクトを完了済みにする
  scenario "user completes a project"
  # ここまで
end
```

ファイルを保存し、bundle exec rspec spec/system/projects_spec.rb というコマンドでスペックを実行してください。<br>
たぶん予想が付いていると思いますが、RSpec は次のような フィードバックを返してきます。<br>

```:terminal
Projects
  user creates a new project
  user completes a project (PENDING: Not yet implemented)

Pending: (Failures listed here are expected and do not affect your suite's status)

  1) Projects user completes a project
     # Not yet implemented
     # ./spec/system/projects_spec.rb:30


Finished in 1.01 seconds (files took 2.9 seconds to load)
2 examples, 0 failures, 1 pending
```

新しいシナリオにいくつかステップを追加しましょう。ここではユーザーがプロジェクトを完了済みにする流れを記述します。<br>
最初に、何が必要で、ユーザーは何をして、それがど んな結果になるのかを考えましょう。<br>

1. プロジェクトを持ったユーザーが必要で、そのユーザーはログインしていないといけない。<br>

2. ユーザーはプロジェクト画面を開き完了(complete) ボタンをクリックする。<br>

3. プロジェクトは完了済み(completed)としてマークされる。<br>

私はときどきテストに必要な情報をコメントとして書き始めます。<br>
こうすれば簡単にコメントをテストコードで置き換えられるからです。<br>

+ `spec/system/projects_spec.rb`を編集<br>

```rb:projects_spec.rb
require 'rails_helper'

RSpec.describe "Projects", type: :system do
  # include LoginSupport

  # ユーザーは新しいプロジェクトを作成する
  scenario "user creates a new project" do
    user = FactoryBot.create(:user)
    # この章で独自に定義したログインヘルパーを使う場合
    # sign_in user
    # もしくは Deviseが提供しているヘルバーを使う場合
    sign_in user
    visit root_path

    expect {
      click_link "New Project"
      fill_in "Name", with: "Test Project"
      fill_in "Description", with: "Trying out Capybara"
      click_button "Create Project"
    }.to change(user.projects, :count).by(1)

    aggregate_failures do
      expect(page).to have_content "Project was successfully created"
      expect(page).to have_content "Test Project"
      expect(page).to have_content "Owner: #{user.name}"
    end
  end

  # ユーザーはプロジェクトを完了済みにする
  # 編集
  scenario "user completes a project" do
    # プロジェクトをもったユーザーを準備する
    # ユーザーはログインしている
    # ユーザーがプロジェクト画面を開き、
    # "complete"ボタンをクリックすると、
    # プロジェクトは完了済みとしてマークされる
  end
  # ここまで
end
```

+ `spec/system/projects_spec.rb`を編集(P203〜)<br>

```rb:projects_spec.rb
require 'rails_helper'

RSpec.describe "Projects", type: :system do
  # include LoginSupport

  # ユーザーは新しいプロジェクトを作成する
  scenario "user creates a new project" do
    user = FactoryBot.create(:user)
    # この章で独自に定義したログインヘルパーを使う場合
    # sign_in user
    # もしくは Deviseが提供しているヘルバーを使う場合
    sign_in user
    visit root_path

    expect {
      click_link "New Project"
      fill_in "Name", with: "Test Project"
      fill_in "Description", with: "Trying out Capybara"
      click_button "Create Project"
    }.to change(user.projects, :count).by(1)

    aggregate_failures do
      expect(page).to have_content "Project was successfully created"
      expect(page).to have_content "Test Project"
      expect(page).to have_content "Owner: #{user.name}"
    end
  end

  # ユーザーはプロジェクトを完了済みにする
  # 編集
  scenario "user completes a project", focus: true do
    user = FactoryBot.create(:user)
    project = FactoryBot.create(:project, owner: user)
    sign_in user

    visit project_path(project)
    click_button "Complete"

    expect(project.reload.completed?).to be true
    expect(page).to \
      have_content "Congratulations, this project is complete!"
    expect(page).to have_content "Completed"
    expect(page).to_not have_button "Complete"
  end
  # ここまで
end
```

`＊`<br>
ここで私は focus: true タグを追加しています。これは bundle exec rspec でこのスペックだけを実行するためです。<br>
この新機能に取り組んでいる間は他にもこのタグを付けるかもしれません。<br>
こうすれば毎回テストスイート全体を実行せずに済みます。みなさんも同じようにする場合は、変更点をコミットする前にタグを削除して、テストスイート全体を実行するのを忘れないようにしてください。<br>
タグを使ってテストの実行スピードを上げる方法については、第9章を参照してください。focus: true の省略形として、:focus を使うこともできます。<br>

この新機能を実現するためのアプリケーションコードはまだ実際に書いていませんが、どのような形で動くのかはすでに記述しています。<br>
まず、Complete と書かれたボタンがあります。このボタンをクリックするとプロジェクトの completed 属性が更新され、値が false から true に変わります。<br>
それからフラッシュメッセージでプロジェクトが完了済みになったことをユーザーに通知します。さらにボタンがなくなるかわりにプロジェクトが完了済みになったことを示すラベルが表示されることを確認します。<br>
これがテスト駆動開発の基本です。テストから先に書き始めることで、コードがどのように振る舞うのかを積極的に考えることができます。<br>

## レッドからグリーンへ

スペックをもう一度実行してください。テストは失敗します! <br>
ですが、テスト駆動開発の場合、これは良いこととされているので覚えておきましょう。<br>
なぜなら、テストが失敗することで次に作業すべきゴールがわかるからです。RSpec は失敗した内容をわかりやすく表示してくれます。<br>

```:terminal
Projects
  user completes a project (FAILED - 1)

Failures:

  1) Projects user completes a project
     Failure/Error: click_button "Complete"

     Capybara::ElementNotFound:
       Unable to find button "Complete" that is not disabled
     # ./spec/system/projects_spec.rb:36:in `block (2 levels) in <top (required)>'

Finished in 0.74369 seconds (files took 2.63 seconds to load)
1 example, 1 failure

Failed examples:
```

さて、この状況を前進させる一番シンプルな方法は何でしょうか? view に新しいボタンを追加してみるとどうなるでしょう?<br>

+ `app/views/projects/_project.html.erb`を編集(p204〜)<br>

```html:_project.html.erb
<h1 class="heading">
  <%= project.name %>
  <%= link_to edit_project_path(project), class: "btn btn-light btn-sm btn-inline" do %>
    <span class="bi bi-pencil-fill" aria-hidden="true"></span>
    Edit
  <% end %>
  <!-- 追加 -->
  <button class="btn btn-light btn-sm btn-inline">
    <span class="bi bi-check-lg" aria-hidden="true">
      Complete
    </span>
  </button>
  <!-- ここまで -->
</h1>

<div class="project-description">
  <%= simple_format project.description %>
</div>

<p>
  Owner:
  <%= @project.owner.name %>
</p>

<p>
  Due:
  <%= full_date(project.due_on) %>
</p>

<hr class="divider">

<h2 class="heading">
  Tasks
  <%= link_to new_project_task_path(@project), class: "btn btn-light btn-sm btn-inline" do %>
    <span class="bi bi-plus-lg" aria-hidden="true"></span>
    Add Task
  <% end %>
</h2>

<table class="table">
  <thead>
    <tr>
      <th>Name</th>
      <th class="task-actions">Actions</th>
    </tr>
  </thead>

  <tbody>
    <%= render @project.tasks %>
  </tbody>
</table>

<hr class="divider">

<div class="row">
  <div class="col-xs-12 col-md-8">
    <h2 class="heading" style="padding: 0 !important; margin: 0 !important">
      Notes
      <%= link_to new_project_note_path(@project), class: "btn btn-light btn-sm btn-inline" do %>
        <span class="bi bi-plus-lg" aria-hidden="true"></span>
        Add Note
      <% end %>
    </h2>
  </div>

  <div class="col-xs-12 col-md-4" style="text-align: right;">
    <%= form_with url: project_notes_path(@project),
                  method: :get, class: "form-inline" do |f| %>
      <div class="input-group mb-3">
        <%= f.search_field :term, class: "form-control", id: :term, placeholder: "Search Notes" %>
        <%= button_tag class: "btn btn-outline-secondary" do %>
          <i class="bi bi-search form-control-feedback"></i>
        <% end %>
      </div>
    <% end %>
  </div>
</div>

<%= render @project.notes %>
```

テストをもう一度実行し、前に進んだことを確認しましょう。<br>

```:terminal
Projects
  user completes a project (FAILED - 1)

Failures:

  1) Projects user completes a project
     Failure/Error: expect(project.reload.completed?).to be true

     NoMethodError:
       undefined method `completed?' for #<Project id: 1, name: "Test Project 1", description: "Sample project for testing purposes", due_on: "2022-11-18", created_at: "2022-11-11 07:48:37.733038000 +0000", updated_at: "2022-11-11 07:48:37.733038000 +0000", user_id: 1>
     # ./spec/system/projects_spec.rb:38:in `block (2 levels) in <top (required)>'

Finished in 0.61124 seconds (files took 2.57 seconds to load)
1 example, 1 failure
```

RSpec は次に何をすべきか、ヒントを与えてくれています。Project モデルに completed? という名前のメソッドを追加してください。<br>
アプリケーションの内容とビジネスロジックによりますが、ここではいくつかの検討事項が浮かび上がってきます。<br>
プロジェクト内のタスクがすべて完了したら、プロジェクトは完了済みになるのでしょうか?もしそうであれば、complete? メソッドをProject モデルに定義し、プロジェクト内の未完了タスクが空(つまり、プロジェクトは完了済み)か、そうでないか(プロジェクトは未完了の意味)を返すことができます。<br>
ただし、今回は完了済みかどうかは、ボタンをクリックするというユーザーの操作によって決定され、完了済みかどうかのステータスは永続化されることになります。<br>
なので、この値を保存するために、新しい Active Record の属性をモデルに追加する必要があるようです。<br>
projects テーブルにこのカラムを追加するマイグレーションを作成し、それを実行してください。<br>

+ `$ bin/rails g migration add_completed_to_projects completed:boolean`を実行<br>

+ `$ bin/rails db:migrate`を実行<br>

+ `＊`<br>
  Rails は自動的に新しいマイグレーションをテストデータベースに適用しようとしますが、毎回成功するとは限りません。<br>
  テストデータベースに対してマイグレーショ ンをまず実行してくださいというエラーメッセージが出た場合は、表示されているメッセージに従い(bin/rails db:migrate RAILS_ENV=test を実行します)、再度テストを実行してください。<br>

+ `＊`<br>
  bin/rails db:migrate の実行時 に “SQLite3::SQLException: duplicate column name: completed” というエラーが発生した場合は、bin/rails db:reset コマンドでデータベースをリセットしてから再度 bin/rails db:migrate を実行してください。<br>

変更を適用したら、新しいスペックをもう一度実行してください。今度は別の理由で失敗しますが、新しい失敗は私たちが前進していることを意味しています。<br>

+ `$ bundle exec rspec spec/system/projects_spec.rb`を実行<br>

```:terminal
Projects
  user completes a project (FAILED - 1)

Failures:

  1) Projects user completes a project
     Failure/Error: expect(project.reload.completed?).to be true

       expected true
            got false
     # ./spec/system/projects_spec.rb:38:in `block (2 levels) in <top (required)>'

Finished in 0.69159 seconds (files took 5.45 seconds to load)
1 example, 1 failure
```

ある意味残念なことですが、この高レベルなテストは他のテストと同じように重要な情報を示しています。<br>
みなさんは何が起きているのかもうわかったかもしれませんが、第6章で説明した Launchy を使ってチェックしてみましょう。<br>
一時的に Launchy をテストに組み込んでください。<br>

+ `spec/system/projects_spec.rb`を編集(P206〜)<br>

```rb:projects_spec.rb
require 'rails_helper'

RSpec.describe "Projects", type: :system do
  # include LoginSupport

  # ユーザーは新しいプロジェクトを作成する
  scenario "user creates a new project" do
    user = FactoryBot.create(:user)
    # この章で独自に定義したログインヘルパーを使う場合
    # sign_in user
    # もしくは Deviseが提供しているヘルバーを使う場合
    sign_in user
    visit root_path

    expect {
      click_link "New Project"
      fill_in "Name", with: "Test Project"
      fill_in "Description", with: "Trying out Capybara"
      click_button "Create Project"
    }.to change(user.projects, :count).by(1)

    aggregate_failures do
      expect(page).to have_content "Project was successfully created"
      expect(page).to have_content "Test Project"
      expect(page).to have_content "Owner: #{user.name}"
    end
  end

  # ユーザーはプロジェクトを完了済みにする
  scenario "user completes a project", focus: true do
    user = FactoryBot.create(:user)
    project = FactoryBot.create(:project, owner: user)
    login_as user, scope: :user # 編集

    visit project_path(project)
    click_button "Complete"
    save_and_open_page # 追加
    expect(project.reload.completed?).to be true
    expect(page).to \
      have_content "Congratulations, this project is complete!"
    expect(page).to have_content "Completed"
    expect(page).to_not have_button "Complete"
  end
end
```

興味深いことに、画面がプロジェクト画面から変わっていません。ボタンをクリックしても何も起きないようですね。<br>
ああそうだ、先ほど view に <button> タグを追加しましたが、このボタンが機能するようにしなければならないのでした。<br>
view を変更してちゃんと動くよ うにしていきましょう。<br>

ですが、ここで新たに設計上の判断が必要になります。データベースに変更を加えるこのボタンのルーティングはどのようにするのが一番良いでしょうか? コードをシンプルに保ち、Project コントローラの update アクションを再利用することもできますが、ここではフラッシュメッセージが異なるため、まったく同じ振る舞いにはなりません。<br>
そこでコントローラに新しいメンバーアクションを追加することにしましょう。<br>
この新機能のテストがいったんパスすれば、時間が許す限り別の実装を試すことも可能です。<br>

この場合は高いレベルから始めて、だんだん下へ降りていくアプローチが良いと思います。<br>
view に戻ってボタンを修正しましょう。<button> タグは Rails の button_to ヘルパーの呼び出しに置き換えてください。<br>

+ `app/views/projects/_project.html.erb`を編集(P207〜)<br>

```html:_project.html.erb
<h1 class="heading">
  <%= project.name %>
  <%= link_to edit_project_path(project), class: "btn btn-light btn-sm btn-inline" do %>
    <span class="bi bi-pencil-fill" aria-hidden="true"></span>
    Edit
  <% end %>
  <!-- 編集 -->
  <%= button_to complete_project_path(project),
    method: :patch,
    form: { style: "display: inline-block;" },
    class: "btn btn-light btn-sm btn-inline" do %>
    <span class="bi bi-chek-lg" aria-hidden="true"></span>
    Complete
  <% end %>
  <!-- ここまで -->
</h1>

<div class="project-description">
  <%= simple_format project.description %>
</div>

<p>
  Owner:
  <%= @project.owner.name %>
</p>

<p>
  Due:
  <%= full_date(project.due_on) %>
</p>

<hr class="divider">

<h2 class="heading">
  Tasks
  <%= link_to new_project_task_path(@project), class: "btn btn-light btn-sm btn-inline" do %>
    <span class="bi bi-plus-lg" aria-hidden="true"></span>
    Add Task
  <% end %>
</h2>

<table class="table">
  <thead>
    <tr>
      <th>Name</th>
      <th class="task-actions">Actions</th>
    </tr>
  </thead>

  <tbody>
    <%= render @project.tasks %>
  </tbody>
</table>

<hr class="divider">

<div class="row">
  <div class="col-xs-12 col-md-8">
    <h2 class="heading" style="padding: 0 !important; margin: 0 !important">
      Notes
      <%= link_to new_project_note_path(@project), class: "btn btn-light btn-sm btn-inline" do %>
        <span class="bi bi-plus-lg" aria-hidden="true"></span>
        Add Note
      <% end %>
    </h2>
  </div>

  <div class="col-xs-12 col-md-4" style="text-align: right;">
    <%= form_with url: project_notes_path(@project),
                  method: :get, class: "form-inline" do |f| %>
      <div class="input-group mb-3">
        <%= f.search_field :term, class: "form-control", id: :term, placeholder: "Search Notes" %>
        <%= button_tag class: "btn btn-outline-secondary" do %>
          <i class="bi bi-search form-control-feedback"></i>
        <% end %>
      </div>
    <% end %>
  </div>
</div>

<%= render @project.notes %>
```

スタイルの記述を少し追加したことに加えて、この新しいアクションで呼ばれるルートヘルパーの定義も考えてみました。<br>
それが complete_project_path です。save_and_open_page をスペックから削除し、テストをもう一度実行してください。<br>
おっと、新しい失敗メッセージが出ました。<br>

```:terminal
Projects
  user completes a project (FAILED - 1)

Failures:

  1) Projects user completes a project
     Failure/Error: <%= button_to complete_project_path(project),

     ActionView::Template::Error:
       undefined method `complete_project_path' for #<ActionView::Base:0x0000000000fd20>
       Did you mean?  compute_asset_path
```

OK、私たちは別に compute_asset_path を使おうとしたわけではありません (訳注:エラ ーメッセージの最後に「もしかして compute_asset_path を使おうとしましたか?」と出力されています)。<br>
ですが、これはまさにこれから使おうとしている新しいルーティングをまだ定義していないというヒントになっています。<br>
では、routes ファイルにルーティングを追加しましょう。<br>

+ `config/routes.rb`を編集<br>

```rb:routes.rb
Rails.application.routes.draw do

  devise_for :users, controllers: { registrations: 'registrations', sessions: 'sessions' }

  authenticated :user do
    root 'projects#index', as: :authenticated_root
  end

  resources :projects do
    resources :notes
    resources :tasks do
      member do
        patch :toggle
      end
    end
    # 追加
    member do
      patch :complete
    end
    # ここまで
  end

  namespace :api do
    resources :projects#, only: [:index, :show, :create]
  end

  root "home#index"
end
```

ルーティングを追加してからスペックを実行すると、別の新しい失敗メッセージが表示されます。<br>

```:terminal
Projects
  user completes a project (FAILED - 1)

Failures:

  1) Projects user completes a project
     Failure/Error: click_button "Complete"

     AbstractController::ActionNotFound:
       The action 'complete' could not be found for ProjectsController
```

RSpec は私たちに何を修正すればいいのか、良いヒントを与えてくれています(訳注:エ ラーメッセージの最後に「ProjectsController に ‘complete’ アクションが見つかりません」と出力されています)。<br>
ここに書かれているとおり、Project コントローラに空の complete アクションを作成しましょう。<br>
追加する場所は Rails のジェネレータで作成された既存の destroy アクションの下にします。<br>

+ `app/controllers/projects_controller.rb`を編集<br>

```rb:projects_controller.rb
class ProjectsController < ApplicationController
  before_action :set_project, only: %i[ show edit update destroy ]
  before_action :project_owner?, except: %i[ index new create ]

  # GET /projects or /projects.json
  def index
    @projects = current_user.projects
  end

  # GET /projects/1 or /projects/1.json
  def show
  end

  # GET /projects/new
  def new
    @project = Project.new
  end

  # GET /projects/1/edit
  def edit
  end

  # POST /projects or /projects.json
  def create
    @project = current_user.projects.new(project_params)

    respond_to do |format|
      if @project.save
        format.html { redirect_to project_url(@project), notice: "Project was successfully created." }
        format.json { render :show, status: :created, location: @project }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @project.errors, status: :unprocessable_entity }
      end
    end
  end

  # PATCH/PUT /projects/1 or /projects/1.json
  def update
    respond_to do |format|
      if @project.update(project_params)
        format.html { redirect_to project_url(@project), notice: "Project was successfully updated." }
        format.json { render :show, status: :ok, location: @project }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: @project.errors, status: :unprocessable_entity }
      end
    end
  end

  # DELETE /projects/1 or /projects/1.json
  def destroy
    @project.destroy

    respond_to do |format|
      format.html { redirect_to projects_url, notice: "Project was successfully destroyed.", status: :see_other }
      format.json { head :no_content }
    end
  end

  # 追加
  def complete

  end
  # ここまで

  private
    # Use callbacks to share common setup or constraints between actions.
    def set_project
      @project = Project.find(params[:id])
    end

    # Only allow a list of trusted parameters through.
    def project_params
      params.require(:project).permit(:name, :description, :due_on)
    end
end
```

中身も書き始めたいという誘惑に駆られますが、テスト駆動開発の信条は「テストを前進させる必要最小限のコードを書く」です。<br>
先ほど、テストはアクションが見つからないと訴えていました。私たちはそれを追加しました。<br>
テストを実行して、現在の状況を確認してみましょう。<br>

```terminal
  1) Projects user completes a project
     Failure/Error: unless @project.owner == current_user

     NoMethodError:
       undefined method `owner' for nil:NilClass

           unless @project.owner == current_user
                          ^^^^^^
     # ./app/controllers/application_controller.rb:11:in `project_owner?'
```

ちょっと面白い結果になりました。まったく別のコントローラでテストが失敗しています!<br>
これはなぜでしょうか? Project コントローラではアクションを実行する前にいくつかのコールバックを設定しています。<br>
ですが、その中に新しく作った complete アクションを含めていませんでした。これが失敗の原因です。<br>
というわけで、アクションを追加しましょう。<br>

+ `app/controllers/projects_controller.rb`を編集(P209〜)<br>

```rb:projects_controller.rb
class ProjectsController < ApplicationController
  before_action :set_project, only: %i[ show edit update destroy complete ] # completeを追加
  before_action :project_owner?, except: %i[ index new create ]

  # GET /projects or /projects.json
  def index
    @projects = current_user.projects
  end

  # GET /projects/1 or /projects/1.json
  def show
  end

  # GET /projects/new
  def new
    @project = Project.new
  end

  # GET /projects/1/edit
  def edit
  end

  # POST /projects or /projects.json
  def create
    @project = current_user.projects.new(project_params)

    respond_to do |format|
      if @project.save
        format.html { redirect_to project_url(@project), notice: "Project was successfully created." }
        format.json { render :show, status: :created, location: @project }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @project.errors, status: :unprocessable_entity }
      end
    end
  end

  # PATCH/PUT /projects/1 or /projects/1.json
  def update
    respond_to do |format|
      if @project.update(project_params)
        format.html { redirect_to project_url(@project), notice: "Project was successfully updated." }
        format.json { render :show, status: :ok, location: @project }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: @project.errors, status: :unprocessable_entity }
      end
    end
  end

  # DELETE /projects/1 or /projects/1.json
  def destroy
    @project.destroy

    respond_to do |format|
      format.html { redirect_to projects_url, notice: "Project was successfully destroyed.", status: :see_other }
      format.json { head :no_content }
    end
  end

  def complete

  end

  private
    # Use callbacks to share common setup or constraints between actions.
    def set_project
      @project = Project.find(params[:id])
    end

    # Only allow a list of trusted parameters through.
    def project_params
      params.require(:project).permit(:name, :description, :due_on)
    end
end
```

ではスペックを再実行してください。<br>

```:terminal
Projects
  user completes a project (FAILED - 1)

Failures:

  1) Projects user completes a project
     Failure/Error: expect(project.reload.completed?).to be true
```

この結果は一見、何歩か後退してしまったように見えます。この失敗メッセージは少し前にも見たんじゃないでしょうか? <br>
ですが実際にはゴールに近づいています。上の失敗メッセージはコントローラのアクションには到達したものの、そのあとに何も起きなかったことを意味しています。<br>
というわけで、正常系のシナリオを満足させるコードを書いていきましょう。<br>
ここでは何の問題もなくユーザーがプロジェクトを完了済みにできることを前提とします。<br>

+ `app/controllers/projects_controller.rb`を編集(P210〜)<br>

```rb:projects_controller.rb
class ProjectsController < ApplicationController
  before_action :set_project, only: %i[ show edit update destroy complete ]
  before_action :project_owner?, except: %i[ index new create ]

  # GET /projects or /projects.json
  def index
    @projects = current_user.projects
  end

  # GET /projects/1 or /projects/1.json
  def show
  end

  # GET /projects/new
  def new
    @project = Project.new
  end

  # GET /projects/1/edit
  def edit
  end

  # POST /projects or /projects.json
  def create
    @project = current_user.projects.new(project_params)

    respond_to do |format|
      if @project.save
        format.html { redirect_to project_url(@project), notice: "Project was successfully created." }
        format.json { render :show, status: :created, location: @project }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @project.errors, status: :unprocessable_entity }
      end
    end
  end

  # PATCH/PUT /projects/1 or /projects/1.json
  def update
    respond_to do |format|
      if @project.update(project_params)
        format.html { redirect_to project_url(@project), notice: "Project was successfully updated." }
        format.json { render :show, status: :ok, location: @project }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: @project.errors, status: :unprocessable_entity }
      end
    end
  end

  # DELETE /projects/1 or /projects/1.json
  def destroy
    @project.destroy

    respond_to do |format|
      format.html { redirect_to projects_url, notice: "Project was successfully destroyed.", status: :see_other }
      format.json { head :no_content }
    end
  end

  def complete
    # 追加
    @project.update!(completed: true)
    redirect_to @project,
    notice: "Congratulations, this project is complete!"
    # ここまで
  end

  private
    # Use callbacks to share common setup or constraints between actions.
    def set_project
      @project = Project.find(params[:id])
    end

    # Only allow a list of trusted parameters through.
    def project_params
      params.require(:project).permit(:name, :description, :due_on)
    end
end
```

スペックをもう一度実行してください。もうそろそろ終わりに近づいてきているはずです!<br>

```:terminal
Projects
  user completes a project (FAILED - 1)

Failures:

  1) Projects user completes a project
     Failure/Error: expect(page).to have_content "Completed"
       expected to find text "Completed" in "Project Manager\nProjects\nAaron Summer Sign Out\nCongratulations, this project is complete!\nTest Project 1 Edit\nComplete\nSample project for testing purposes\nOwner: Aaron Summer\nDue: November 18, 2022 (6 days from now)\nTasks Add Task\nName Actions\nNotes Add Note"
```

ここに表示されている画面内の文言を読んでいくと、どうやら Completed という文字列が画面に出力されていないようです。<br>
これはまだこの文字列を追加していないからであり、さらに言うと私たちがやっているのはテスト駆動開発だからです。<br>
では view に文字列を追加してみましょう。先ほど追加したボタンの隣によく目立つラベルの <span> タグを追加してください。<br>

+ `pp/views/projects/_project.html.erb`を編集(P211〜)<br>

```html:_project.html.erb
<h1 class="heading">
  <%= project.name %>
  <%= link_to edit_project_path(project), class: "btn btn-light btn-sm btn-inline" do %>
    <span class="bi bi-pencil-fill" aria-hidden="true"></span>
    Edit
  <% end %>
  <%= button_to complete_project_path(project),
    method: :patch,
    form: { style: "display: inline-block;" },
    class: "btn btn-light btn-sm btn-inline" do %>
    <span class="bi bi-chek-lg" aria-hidden="true"></span>
    Complete
  <% end %>
  <span class="badge bg-success">Completed</span> <!-- 追加 -->
</h1>

<div class="project-description">
  <%= simple_format project.description %>
</div>

<p>
  Owner:
  <%= @project.owner.name %>
</p>

<p>
  Due:
  <%= full_date(project.due_on) %>
</p>

<hr class="divider">

<h2 class="heading">
  Tasks
  <%= link_to new_project_task_path(@project), class: "btn btn-light btn-sm btn-inline" do %>
    <span class="bi bi-plus-lg" aria-hidden="true"></span>
    Add Task
  <% end %>
</h2>

<table class="table">
  <thead>
    <tr>
      <th>Name</th>
      <th class="task-actions">Actions</th>
    </tr>
  </thead>

  <tbody>
    <%= render @project.tasks %>
  </tbody>
</table>

<hr class="divider">

<div class="row">
  <div class="col-xs-12 col-md-8">
    <h2 class="heading" style="padding: 0 !important; margin: 0 !important">
      Notes
      <%= link_to new_project_note_path(@project), class: "btn btn-light btn-sm btn-inline" do %>
        <span class="bi bi-plus-lg" aria-hidden="true"></span>
        Add Note
      <% end %>
    </h2>
  </div>

  <div class="col-xs-12 col-md-4" style="text-align: right;">
    <%= form_with url: project_notes_path(@project),
                  method: :get, class: "form-inline" do |f| %>
      <div class="input-group mb-3">
        <%= f.search_field :term, class: "form-control", id: :term, placeholder: "Search Notes" %>
        <%= button_tag class: "btn btn-outline-secondary" do %>
          <i class="bi bi-search form-control-feedback"></i>
        <% end %>
      </div>
    <% end %>
  </div>
</div>

<%= render @project.notes %>
```

テストをもう一度実行してください。失敗しているのは最後のステップだけです!<br>

```:terminal
Projects
  user completes a project (FAILED - 1)

Failures:

  1) Projects user completes a project
     Failure/Error: expect(page).to_not have_button "Complete"
       expected not to find visible button "Complete" that is not disabled, found 1 match: "Complete"
```

プロジェクトは完了済みになっていますが、Complete ボタンがまだ表示されたままです。<br>
このままだとユーザーを混乱させてしまうので、完了済みのプロジェクトを開いたときは UI にボタンを表示しないようにすべきです。<br>
view に新たな変更を加えれば、これを実現できます。<br>

+ `app/views/projects/_project.html.erb`を編集(P212〜)<br>

```html:_project.html.erb
<h1 class="heading">
  <%= project.name %>
  <%= link_to edit_project_path(project), class: "btn btn-light btn-sm btn-inline" do %>
    <span class="bi bi-pencil-fill" aria-hidden="true"></span>
    Edit
  <% end %>
  <% unless project.completed? %> <!-- 追加 -->
    <%= button_to complete_project_path(project),
    method: :patch,
    form: { style: "display: inline-block;" },
    class: "btn btn-light btn-sm btn-inline" do %>
      <span class="bi bi-chek-lg" aria-hidden="true"></span>
      Complete
    <% end %>
  <% end %> <!-- 追加 -->
  <span class="badge bg-success">Completed</span>
</h1>

<div class="project-description">
  <%= simple_format project.description %>
</div>

<p>
  Owner:
  <%= @project.owner.name %>
</p>

<p>
  Due:
  <%= full_date(project.due_on) %>
</p>

<hr class="divider">

<h2 class="heading">
  Tasks
  <%= link_to new_project_task_path(@project), class: "btn btn-light btn-sm btn-inline" do %>
    <span class="bi bi-plus-lg" aria-hidden="true"></span>
    Add Task
  <% end %>
</h2>

<table class="table">
  <thead>
    <tr>
      <th>Name</th>
      <th class="task-actions">Actions</th>
    </tr>
  </thead>

  <tbody>
    <%= render @project.tasks %>
  </tbody>
</table>

<hr class="divider">

<div class="row">
  <div class="col-xs-12 col-md-8">
    <h2 class="heading" style="padding: 0 !important; margin: 0 !important">
      Notes
      <%= link_to new_project_note_path(@project), class: "btn btn-light btn-sm btn-inline" do %>
        <span class="bi bi-plus-lg" aria-hidden="true"></span>
        Add Note
      <% end %>
    </h2>
  </div>

  <div class="col-xs-12 col-md-4" style="text-align: right;">
    <%= form_with url: project_notes_path(@project),
                  method: :get, class: "form-inline" do |f| %>
      <div class="input-group mb-3">
        <%= f.search_field :term, class: "form-control", id: :term, placeholder: "Search Notes" %>
        <%= button_tag class: "btn btn-outline-secondary" do %>
          <i class="bi bi-search form-control-feedback"></i>
        <% end %>
      </div>
    <% end %>
  </div>
</div>

<%= render @project.notes %>
```

スペックを実行してください。・・・ついにパスしました!<br>

```:terminal
Projects
  user completes a project

Finished in 0.78579 seconds (files took 2.78 seconds to load)
1 example, 0 failures
```

私は view のロジックに変更が入ったらブラウザでどうなったのか確認するようにしています(API のエンドポイントを書いているときは curl かその他の HTTP クライアントでテス トします)。<br>
Rails の開発用サーバーが動いていなければ、サーバーを起動してください。それからプロジェクト画面をブラウザで開いてください。<br>
Complete ボタンをクリックすると本当にプロジェクトが完了済みになり、ボタンが表示されなくなることを確認してください・・・ と言いたいところですが、プロジェクトの完了状態にかかわらず、先ほど追加した Completed のラベルが表示されています!<br>
どうやらこの新機能が完成したと宣言する前に、テストをちょっと変更した方がいいようです。<br>
新しいアクションを実行する前に、Completed のラベルが画面に表示されていないことを確認しましょう。<br>

+ `spec/system/projects_spec.rb`を編集(P213〜)<br>

```rb:projects_spec.rb
require 'rails_helper'

RSpec.describe "Projects", type: :system do
  # include LoginSupport

  # ユーザーは新しいプロジェクトを作成する
  scenario "user creates a new project" do
    user = FactoryBot.create(:user)
    # この章で独自に定義したログインヘルパーを使う場合
    # sign_in user
    # もしくは Deviseが提供しているヘルバーを使う場合
    sign_in user
    visit root_path

    expect {
      click_link "New Project"
      fill_in "Name", with: "Test Project"
      fill_in "Description", with: "Trying out Capybara"
      click_button "Create Project"
    }.to change(user.projects, :count).by(1)

    aggregate_failures do
      expect(page).to have_content "Project was successfully created"
      expect(page).to have_content "Test Project"
      expect(page).to have_content "Owner: #{user.name}"
    end
  end

  # ユーザーはプロジェクトを完了済みにする
  # 編集
  scenario "user completes a project" do
    user = FactoryBot.create(:user)
    project = FactoryBot.create(:project, owner: user)
    login_as user, scope: :user

    visit project_path(project)

    expect(page).to_not have_content "Completed"

    click_button "Complete"
    # ここまで

    expect(project.reload.completed?).to be true
    expect(page).to \
      have_content "Congratulations, this project is complete!"
    expect(page).to have_content "Completed"
    expect(page).to_not have_button "Complete"
  end
end
```

こうするとスペックはまた失敗してしまいます。ですが、これは一時的なものです。<br>

```:terminal
Failures:

  1) Projects user completes a project
     Failure/Error: expect(page).to_not have_content "Completed"
       expected not to find text "Completed" in "Project Manager\nProjects\nAaron Summer Sign Out\nTest Project 27 Edit\nComplete\nCompleted\nSample project for testing purposes\nOwner: Aaron Summer\nDue: November 22, 2022 (6 days from now)\nTasks Add Task\nName Actions\nNotes Add Note"
```

ボタンの周りに付けていた条件分岐を変更し、ラベルの表示も制御するようにしましょう。<br>

+ `app/views/projects/_project.html.erb`を編集(P214〜)<br>

```html:_project.html.erb
<h1 class="heading">
  <%= project.name %>
  <%= link_to edit_project_path(project), class: "btn btn-light btn-sm btn-inline" do %>
    <span class="bi bi-pencil-fill" aria-hidden="true"></span>
    Edit
  <% end %>
  <!-- 編集 -->
  <% if project.completed? %>
    <span class="badge bg-success">Completed</span>
  <% else %>
    <%= button_to complete_project_path(project),
      method: :patch,
      form: { style: "display: inline-block;"},
      class: "btn btn-light btn-sm btn-inline" do %>
      <span class="bi bi-check-lg" aria-hidden="true"></span>
      Complete
    <% end %>
  <% end %>
  <!-- ここまで -->
</h1>

<div class="project-description">
  <%= simple_format project.description %>
</div>

<p>
  Owner:
  <%= @project.owner.name %>
</p>

<p>
  Due:
  <%= full_date(project.due_on) %>
</p>

<hr class="divider">

<h2 class="heading">
  Tasks
  <%= link_to new_project_task_path(@project), class: "btn btn-light btn-sm btn-inline" do %>
    <span class="bi bi-plus-lg" aria-hidden="true"></span>
    Add Task
  <% end %>
</h2>

<table class="table">
  <thead>
    <tr>
      <th>Name</th>
      <th class="task-actions">Actions</th>
    </tr>
  </thead>

  <tbody>
    <%= render @project.tasks %>
  </tbody>
</table>

<hr class="divider">

<div class="row">
  <div class="col-xs-12 col-md-8">
    <h2 class="heading" style="padding: 0 !important; margin: 0 !important">
      Notes
      <%= link_to new_project_note_path(@project), class: "btn btn-light btn-sm btn-inline" do %>
        <span class="bi bi-plus-lg" aria-hidden="true"></span>
        Add Note
      <% end %>
    </h2>
  </div>

  <div class="col-xs-12 col-md-4" style="text-align: right;">
    <%= form_with url: project_notes_path(@project),
                  method: :get, class: "form-inline" do |f| %>
      <div class="input-group mb-3">
        <%= f.search_field :term, class: "form-control", id: :term, placeholder: "Search Notes" %>
        <%= button_tag class: "btn btn-outline-secondary" do %>
          <i class="bi bi-search form-control-feedback"></i>
        <% end %>
      </div>
    <% end %>
  </div>
</div>

<%= render @project.notes %>
```

そしてスペックをもう一度実行してください。<br>

```:terminal
Projects
  user creates a new project
  user completes a project

Finished in 0.98608 seconds (files took 2.35 seconds to load)
2 examples, 0 failures
```

グリーンに戻りました!<br>
ここまでは一つのスペックだけを実行してきましたが、別のことを始める前はいつもテストスイート全体をチェックするようにすべきです。<br>
bundle exec rspec を実行し、今回の変更が既存の機能を何も壊していないことを確認してください。<br>
TDD の最中に focus: true タグ、もしくは :focus タグを付けていた場合は、それも外すようにしましょう。<br>
テストスイートはグリーンになりました。これなら大丈夫そうです!<br>

## 外から中へ進む（Going outside-in）(P215〜)

私たちは高レベルの統合テストを使い、途中でソフトウェアがどんな振る舞いを持つべきか検討しながら、この新しい機能を完成させました。<br>
この過程の大半において、私たちは次にどんなコードを書くべきか、すぐにわかりました。なぜなら RSpec がその都度私たちにフィードバックを伝えてくれたからです。<br>
ですが、数回は何が起きていたのかじっくり調べる必要もありました。今回のケースでは、Launchy がデバッグツールとして大変役に立ちました。<br>
しかし、問題を理解するために、さらにテストを書かなければいけないこともよくあります。<br>
こうやって追加したテストはあなたの書いたコードをいろんなレベルから突っつき、高レベルのテストだけでは集めてくるのが難しい、有益な情報を提供してくれます。<br>
これがまさに、外から中へ(outside-in) のテストです。私はいつもこのような方法でテスト駆動開発を進めています。<br>
私はブラウザのシミュレートの実行コストが成果に見合わないと思った場合に、低レベルのテストを活用することもよくやります。私たちが書いた統合テストは正常系の操作です。<br>
プロジェクトを完了済みにする際、ユーザーはエラーに遭遇することはありませんでした。では、更新に失敗する場合のテストも高レベルのテストとして新たに追加する必要があるでしょうか? いいえ、これは追加する必要はないかもしれません。<br>
なぜなら私たちはすでに Complete ボタンが正しく実装され、画面にフラッシュメッセージが正しく表示されていることを確認したからです。<br>
この次はコントローラのテストに降りていって、異常系の操作をいろいろとテストするのが良いかもしれません。<br>
たとえば、適切なフラッシュメッセージが設定され、プロジェクトオブジェクトが変わらないことを検証する、といったテストです。<br>
この場合、すでにコードは書いてあるので、コントローラのテストではプロジェクトを完了済みにする際に発生したエラーも適切に処理されることを確認するだけです。<br>
このテストは既存の Project コントローラのスペックに追加できます。<br>

+ `spec/controllers/projects_controller_spec.rb`を編集(P216〜)<br>

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
        aggregate_failures do
          expect(response).to be_successful
          expect(response).to have_http_status "200"
        end
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

  # 追加
  describe "#complete" do
    # 認証済みのユーザーとして
    context "as an authenticated user" do
      let!(:project) { FactoryBot.create(:project, completed: nil) }

      before do
        sign_in project.owner
      end

      # 成功しないプロジェクトの完了
      describe "an unsuccessful comletion" do
        before do
          allow_any_instance_of(Project).
            to receive(:update).
            with(completed: true).
            and_return(false)
        end

        # プロジェクト画面にリダイレクトすること
        it "redirects to the project page" do
          patch :complete, params: { id: project.id }
          expect(response).to redirect_to project_path(project)
        end

        # フラッシュを設定すること
        it "sets the flash" do
          patch :complete, params: { id: project.id }
          expect(flash[:alert]).to eq "Unable to complete project."
        end

        # プロジェクトを完了済みにしないこと
        it "doesn't mark the project as completed" do
          expect {
            patch :complete, params: { id: project.id }
          }.to_not change(project, :completed)
        end
      end
    end
  end
  # ここまで
end
```

上の example はいずれも失敗をシミュレートするのに十分なものです。ここでは allow_any_instance_of を使って、失敗を再現しました。<br>
allow_any_instance_of は第9章で使った allow メソッドの仲間です。<br>
このコードではあらゆる(any) プロジェクトオブジェクトに対する update の呼び出しに割って入り、プロジェクトの完了状態を保存しないようにしています。<br>
allow_any_instance_of は問題を引き起こしやすいメソッドなので、describe "an unsuccessful completion" ブロックの中でだけ使われるように注意しなければなりません。<br>

このテストを実行し、何が起きるか見てみましょう。<br>

+ `$ bundle exec rspec spec/controllers/projects_controller_spec.rb`を実行<br>

```:terminal
Failures:

  1) ProjectsController#complete as an authenticated user an unsuccessful comletion sets the flash
     Failure/Error: expect(flash[:alert]).to eq "Unable to complete project."

       expected: "Unable to complete project."
            got: nil

       (compared using ==)
     # ./spec/controllers/projects_controller_spec.rb:219:in `block (5 levels) in <top (required)>'
```

この結果を見ると、ユーザーは期待どおりにリダイレクトされ、プロジェクトも意図した通り、完了済みになっていないようです。<br>
しかし、問題の発生を知らせるメッセージが設定されていません。<br>
アプリケーションコードに戻り、これを処理する条件分岐を追加しましょう。<br>

+ `app/controllers/projects_controller.rb`を編集(P218〜)<br>

```rb:projects_controller.rb
class ProjectsController < ApplicationController
  before_action :set_project, only: %i[ show edit update destroy complete ]
  before_action :project_owner?, except: %i[ index new create ]

  # GET /projects or /projects.json
  def index
    @projects = current_user.projects
  end

  # GET /projects/1 or /projects/1.json
  def show
  end

  # GET /projects/new
  def new
    @project = Project.new
  end

  # GET /projects/1/edit
  def edit
  end

  # POST /projects or /projects.json
  def create
    @project = current_user.projects.new(project_params)

    respond_to do |format|
      if @project.save
        format.html { redirect_to project_url(@project), notice: "Project was successfully created." }
        format.json { render :show, status: :created, location: @project }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @project.errors, status: :unprocessable_entity }
      end
    end
  end

  # PATCH/PUT /projects/1 or /projects/1.json
  def update
    respond_to do |format|
      if @project.update(project_params)
        format.html { redirect_to project_url(@project), notice: "Project was successfully updated." }
        format.json { render :show, status: :ok, location: @project }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: @project.errors, status: :unprocessable_entity }
      end
    end
  end

  # DELETE /projects/1 or /projects/1.json
  def destroy
    @project.destroy

    respond_to do |format|
      format.html { redirect_to projects_url, notice: "Project was successfully destroyed.", status: :see_other }
      format.json { head :no_content }
    end
  end

  # 編集
  def complete
    if @project.update(completed: true)
      redirect_to @project,
      notice: "Congratulations, this project is complete!"
    else
      redirect_to @project, alert: "Unable to complete project."
    end
  end
  # ここまで

  private
    # Use callbacks to share common setup or constraints between actions.
    def set_project
      @project = Project.find(params[:id])
    end

    # Only allow a list of trusted parameters through.
    def project_params
      params.require(:project).permit(:name, :description, :due_on)
    end
end
```

このように変更すれば、新しいテストは全部パスします。これで正常系も異常系も、どちらもテストすることができました。<br>
ここまでの内容をまとめると、私は「外から中へ」のテストをするときは、高レベルのテストから始めて、ソフトウェアが意図したとおりに動いていることを確認するようにします。<br>
このとき、すべての前提条件とユーザーの入力値は正しいものとします(つまり 正常系 )。今回は統合テストとして、ブラウザのシミュレーションを行うシステムテストの形式を利用しました。<br>
ですが、API のテストをする場合はリクエストスペックを使います(第7章 を参照)。それから、低いレベルのテストに降りていき、可能な限り直接、細かい内容をテストします。<br>
今回はコントローラスペックを使いましたが、たいていの場合、モデルを直接テストしても大丈夫です。<br>
このテクニックはビジネスロジックを普通の Rails のモデルやコントローラから取り出して、サービスオブジェクトのような単体のクラスに移動させるかどうか検討する場合にも使えます。<br>

`＊`<br>
筆者は[テストを使って、サービスオブジェクトやそれに類似したパターンにリファクタリングする方法](https://everydayrails.com/2017/11/20/replace-rspec-controller-tests.html) を、Everyday Rails ブログで説明しています。<br>
ただし、本書の範疇を超えてしまうため、ここでは実施しません。<br>

テストコードからフィードバックをもらい、その内容に従ってください。<br>
もし、今使っているテストから十分なフィードバックが得られない場合は、レベルをもう一段下げてみてください。<br>

## レッド・グリーン・リファクタのサイクル

新機能はひととおり完成しました。ですが、これで終わりではありません。<br>
ちゃんとパスするテストコードを使って、ここからさらに自分が書いたコードを改善することができます。<br>
「レッド・グリーン・リファクタ」でいうところの「リファクタ」の段階に到達したわけです。<br>
新しいテストコードを使えば、他の実装方法を検討したり、先ほど書いたコードをきれいにまとめたりすることができます。<br>
リファクタリングは非常に複雑で、詳しく説明し始めると本書の範疇を超えてしまいます。<br>
ですが、検討すべき選択肢はいくつか存在します。簡単なものから難しいものの順に並べてみましょう。<br>

+ 私たちは projects/_project の view に新しい条件分岐を追加しました。<br>
  この分岐は Complete ボタンを表示するか、もしくはプロジェクトが完了済みであることを表示するかを決めるためのものです。<br>
  私たちはこのコードをパーシャル view に抽出して、メイン view をシンプルに保つべきでしょうか?<br>

+ プロジェクトを完了させるルーティングを新たに実装する際、私たちは違う形でコントローラを構築する方法についても簡単に議論し ました。<br>
  今回選択した実装方法は本当にベストでしょうか? (この機能をテストするために何をやったのか見直してみると、私はちょっと自信がなくなります。コントローラのテストで Active Record のメソッドをスタブ化するのは、コントローラが多くのことをやりすぎているヒントかもしれません。)<br>

+ ビジネスロジックをコントローラから取り出して他の場所に移動させると、テストがよりシンプルになるかもしれません。<br>
  ですが、どこがいいでしょうか? Project モデルに移動して Project#mark_completed のような新しいメソッドを作るとシンプルになりそうです。<br>
  ですが、こういったアプローチを採用しすぎると、巨大なモデルができあがってしまう恐れがあります。<br>
  別のアプローチとして、プロジェクトを完了済みにすることだけに責任を持つ、サービスオブジェクトにロジックを移動させる方法もあります。<br>
  そうすれば、このサービスオブジェクトを直接テストするだけで済むので、コントローラを動かす必要がなくなります。<br>

+ もしくはまったく異なるコントローラの構成を選択することもできます。<br>
  既存のコント ローラにアクションを追加するかわりに、ネストしたリソースフル(resourceful)なコントローラを新たに作り、<br>
  update アクションを定義して親のプロジェクトを完了させるのはどうでしょうか?<br>

別の実装方法を検討する際は、テストを活用してください。さらに、コードを書いていたときに、テストコードが教えてくれたことも思い出してください。<br>
ほとんどの場合、コードを書く方法は一つだけとは限りません。いくつかの選択肢があり、それぞれに⻑所と短所があります。<br>
リファクタリングをするときは、小さくてインクリメンタルな変更を加えていってください。<br>
そして、テストスイートをグリーンに保ってください。これがリファクタリングの鍵となる手順です。どんな変更を加えても、テストは常にパスさせる必要があります。 (もしくは、失敗したとしても、それは一時的なものであるべきです。)<br>
リファクタリングの最中は、高いレベルのテストと低いレベルのテストの間を行ったり来たりするかもしれません。<br>
システムスペックから始まり、モデルやコントローラ、もしくは独立した Ruby オブジェクトへと対象のレベルが下がっていくこともあります。<br>
対象となるテストのレベルは、アプリケーション内のどこにコードを置くともっとも都合が良いのか、そしてどのレベルのテストが最も適切なフィードバックを返してくれるのかによって、変わってくるものです。<br>

## まとめ

以上が RSpec を使って Rails アプリケーションに新しいフィーチャを開発するときの私のやり方です。<br>
紙面で見るとステップがかなり多いように見えるかもしれませんが、実際は短期的な視点で考えてもそれほど大した作業ではありません。<br>
そして⻑期的な視点で考えても、新機能を実装する際にテストを書いて早期にリファクタリングすれば、将来的な時間をかなり節約できます。<br>
さあこれであなたは自分のプロジェクトでも同じように活用できるツールを全部手に入れました!<br>

## 演習問題

+ 私たちはプロジェクトを完了済みにする新機能の最初の要件を一緒に実装しました。<br>
  ですが、ユーザーのダッシュボードから完了済みのプロジェクトを非表示にする要件が残ったままになっています。<br>
  これをテスト駆動開発で実装してみましょう。最初はこんな統合テストから書き始めてください。<br>
  すなわち、完了済みのプロジェクトと未完了のプロジェクトを準備し、それからプロジェクトオーナーのダッシュボードを開いて適切なプロジェクトが表示されるか(または表示されないか)を検証してください。<br>

+ この機能が実装されると、ユーザーはどうやって完了済みのプロジェクトにアクセスするのでしょうか?<br>
  どう動くべきかをまず検討し、それからその仕様を TDD で実装してください。<br>

+ だいたいできたと思ったら、「レッド・グリーン・リファクタ」の項で挙げたリファクタリングの選択肢から、一つ、もしくはそれ以上のアプローチを選択して実験してみてください。<br>
  リファクタリングをすると、テストの性質がどう変わるでしょうか?<br>
  とくに 異常系のテストケースに注目してみてください。<br>
