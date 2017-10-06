# Timecrowdに固定時間を追加するWebアプリ作ってみた
Timecrowdに対し、下記のようなことが出来るAPIを週末にささっと作ってみました。
- input
  - 日時
    - 2017/10/01 〜 2017/10/05,10:00,11:00
  - タスク
    - 朝礼
- output
  - 2017/10/01から2017/10/05の10:00から11:00に「朝礼」タスクを行ったタイムエントリーを作成する

背景としては、毎度発生する固定時間を一括で入力したかったのです。  
需要があるのかわかりませんが、何かの役に立てばいいなと思います。

## Timecrowdとは？
[Timecrowd](https://marketers-store.com/solutions/25)は、リアルタイムにチームメンバーのスケジュール、タスク、
活動を共有することを目的としたアプリケーションです。  
つまり、本来固定時間を一括でガッと入力するような使い方は想定されていません。。  
弊社においては、開発チームの活動実績を報告するために使用されています。

## やりたいこと
Timecrowdに一括して、タスクのタイムエントリーを登録したい。  

## 環境
環境はRuby on Railsを使用しています。  
Gemには、ラフノート株式会社様のomniauth-timecrowdを使用しています。

-  Ruby 2.2.0p0 (2014-12-25 revision 49005) [x86_64-darwin16]
-  Rails 4.2.1
-  [DateTimePicker](https://xdsoft.net/jqplugins/datetimepicker/)  
 
ユーザー入力の補助のため、DateTimePickerを使用しています。  
railsは3000番ポートを使用するようにしています。

## 前準備
Timecrowdにアプリケーションを登録する必要があります。  
[アプリケーション](https://timecrowd.net/oauth/applications)を登録し、.envを作成してください。
なお、コールバックURLは[http://localhost:3000/auth/timecrowd/callback]のようにします。
```.env
TIMECROWD_CLIENT_ID="ID"
TIMECROWD_CLIENT_SECRET="SECRET"
TIMECROWD_SITE="https://timecrowd.net/"
```

### Gemfile
```ruby:Gemfile
source 'https://rubygems.org'

gem 'rails', '4.2.1'
gem 'sqlite3'
gem 'sass-rails', '~> 5.0'
gem 'uglifier', '>= 1.3.0'
gem 'coffee-rails', '~> 4.1.0'
gem 'jquery-rails'
gem 'turbolinks'
gem 'jbuilder', '~> 2.0'
gem 'sdoc', '~> 0.4.0', group: :doc

gem 'haml-rails'
gem 'erb2haml'

group :development, :test do
  gem 'byebug'
  gem 'pry-byebug'
  gem 'Web-console', '~> 2.0'
  gem 'spring'
end

gem 'dotenv-rails'
gem 'omniauth-timecrowd', github: 'ruffnote/omniauth-timecrowd'

```
### TimecrowdAPI
今回使用するAPIについて説明します。  
- [GET][tasks](https://timecrowd.net/apidoc/v1/tasks/index.html)  
タスクの一覧を取得する。
  - api/v1/teams/*TEAM_ID*/tasks?page=*PAGE_NO*
    -  TEAM_ID : Timecrowd上で作成するチームのID
    -  PAGE_NO : APIは最大100個のオブジェクトしか返せないため、それ以上を取得するときにpageを指定する。
- [POST][time_entries](https://timecrowd.net/apidoc/v1/time_entries/create.html)  
タスクに紐付いたタイムエントリーを作成する
  - /api/v1/time_entries
    - Json
    ```ruby:post
     body: {
       task: { title: title,
               team_id: team_id,
               key: key,
               url: url },
       parent: { title: '会社',
                 key: '会社',
                 url: '' }
     })
    ```
    - task
      - title: taskのデータ。タスクのタイトル。
      - key: taskのデータ。タスクのタイトルから記号を除いたものになる。
      - url:  taskのデータ。WebフックのURLが入るっぽい。弊社では空欄。
    - parent  
      基本的にはtaskと同じ。タイムクラウド上、そのタスクが所属する親のタスクを
      指定する。指定しなかった場合、自分と同名の親タスクができてしまう。  
      カテゴリーの最上位のタスクを設定すれば良い。 
      また、親の一覧をAPIで取得することも出来る。
- [PATCH][time_entries](https://timecrowd.net/apidoc/v1/time_entries/update.html)  
タイムエントリーの時間を更新する
  - /api/v1/time_entries/*ID*
    -ID : タイムエントリーのID。POSTで作成した時のレスポンスから取得する。
  - Json
  ```ruby:patch
  body: {
    time_entry: { started_at: started_at.to_i,
                  stopped_at: stopped_at.to_i,
                  time_trackable_id: time_id }
  })
  ```
  - time_entry
    - started_at: タイムエントリーの開始時間。秒で指定し、世界標準時が使用される。
    - stopped_at: タイムエントリーの終了時間。病で指定し、世界標準時が使用される。
    - time_trackable_id: taskのデータ。タイムエントリーを紐付けるタスクのID。

# 全体の流れ
Webアプリケーションの処理の流れはこのようになっています。  
基本のベースは、omniauth-timecrowdのexampleを使用しました。
1. omniauth-timecrowdでアクセストークンを発行できるようにする
2. [GET]tasks でチーム全体のタスクを取得し、viewに表示する。
3. ユーザはviewに表示されたタスクから、入力したいタスクを選ぶ。
4. 期間と時間を入力させるため、viewを表示する。
5. controllerでタスクの情報と、ユーザ入力の情報を受取る。
6. controllerで、4のデータをもとに[POST]time_entriesと、[PATCH]time_entries
を使用して期間入力を行っていく。

## 1.アクセストークンの発行
omniauth-timecrowdを使ってアクセストークンを発行する必要があります。  
そのため、認証用のモジュールを作成します。
initを実行し、成功したらtrueが返り、@access_tokenを使用することができます。
```ruby:auth_client.rb
# TimecrowdのAPIを使用するための認証モジュール
module AuthClient
  def init
    if auth.present?
      @signed_in = true
      @nickname = auth['info']['nickname']
      @image = auth['info']['image']
      oauth
      true
    else
      false
    end
  end

  private

  def auth
    if session['auth'].present?
      session['auth']
    else
      tmp = request.env['omniauth.auth']
      return session['auth'] = tmp.except('extra') if tmp.present?
    end
  end

  def oauth
    token = auth['credentials']['token']

    client = OAuth2::Client.new(ENV['TIMECROWD_CLIENT_ID'], ENV['TIMECROWD_CLIENT_SECRET'], site: ENV['TIMECROWD_SITE'], ssl: { verify: false })
    @access_token = OAuth2::AccessToken.new(client, token)
  end
end
```

## 2. チームのタスクを取得する
タスクを取得するためのcontrollerは、下記の処理を行っています。
1. タスクを全て取得する
1. 不要なタスクをフィルタする
1. タイトルによってソートする
```ruby:welcome_controller
  def set_filtering
    tmp_array = []
    page = 1
    loop do
      tmp = @access_token.get("api/v1/teams/2246/tasks?page=#{page}").parsed
      tmp_array.push(tmp['tasks'])
      break if tmp['is_last_page']
      page += 1
    end
    # task整理
    filters = []
    tmp_array.each do |task_list|
      task_list.each do |task|
        # カテゴリの深さによってフィルタをかけておく
        next if task['ancestry_depth'].zero?
        next if task['ancestry_depth'] == 1
        next if task['ancestry_depth'] == 2
        filters.push(task)
      end
    end

    # titleによってソート
    filters.sort! do |a, b|
      ret = a['title'].casecmp(b['title'])
      ret.zero? ? a['title'] <=> b['title'] : ret
    end
    @tasks = filters
  end
```
## 3. ユーザはタスクを選択する
controllerで表示するタスクが全て取得された後、
viewにそれらを表示します。
下記はomniauth-timecrowdのexampleほぼそのままですが、Taskを全部表示するようにしています。  
タイムエントリーを表示するために必要な情報は、クエリにして送信します。
```haml:index.html.haml
- if @signed_in
  %p
    = image_tag @image, size: '24x24'
    = @nickname
  %ul
    - @tasks.each do |task|
      %li= link_to "#{task['title']} | #{task['ancestry_depth']}", tasks_path({id: task['id'], title: task['title'],team_id: task['team_id'], key: task['key'], url: task['url']}), target: '_blank'
- else
  = link_to 'Sign in', '/auth/timecrowd'
```

## 4.ユーザーに期間を入力させる
ユーザに期間を入力させるために単純なviewを用意します。
簡単に入力させるために、datetimepickerを使用しています。

また、期間については開始期間を入力すると、同値が終了期間に入力されるよう作っています。  
少なくとも、開始期間よりは後になるはずなので、入力の煩わしさが軽減されます。  
datetimepickerで使用しているallowTimesオプションは、選択できる時間を指定するオプションです。
```haml:show.html.haml
:javascript
  function datetime_cnv(id, t){
      year = String(t.getFullYear());
      month = ('00' + String(t.getMonth() + 1)).slice(-2);
      date = ('00' + String(t.getDate())).slice(-2);
      datetime = year+'/'+month + '/' + date;

      $(id).val(datetime);
  }
  $(function(){
    $('#begin1').datetimepicker({
    onClose: function(t){
      datetime_cnv('#end1', t);
    }});

    $('.datepick').datetimepicker({
      timepicker: false,
      format: 'Y/m/d'});

    $('.timepick').datetimepicker({
      allowTimes:['09:00', '10:00', '11:00', '12:00', '13:00',
      '14:00', '15:00', '16:00', '17:00'],
      datepicker: false,
      format: 'H:i'
    });
  });

%div
  = form_tag('tasks/gen', method: 'get') do
    = "id: #{@item['id']}"
    = hidden_field_tag('id', @item['id'])
    %br
    = "team_id: #{@item['team_id']}"
    = hidden_field_tag('team_id', @item['team_id'])
    %br
    = "key: #{@item['key']}"
    = hidden_field_tag('key', @item['key'])
    %br
    %div
      = '期間1: '
      = text_field_tag('begin1', nil, class: 'datepick')
      = ' 〜 '
      = text_field_tag('end1', nil, class: 'datepick')
      = ' 時間1: '
      = text_field_tag('time1', nil, class: 'timepick')
      = ' 〜 '
      = text_field_tag('time_to1', nil, class: 'timepick')
      %br
    = submit_tag('Generate')
```

## 5. controllerでユーザの入力を受け取る
4.で作成した情報は、Controllerのアクションgenで処理されます。  
4.で作成するviewに、最大4つの期間入力を実装しようと思っていました。そのため、separateというメソッドを
作成し、配列をループ処理するように設計しました(適当ですね...)。

```ruby:tasks_controller
  def separate(params)
    ary = {}
    (1..4).each do |val|
      tmp = {}
      tmp['begin'] = params["begin#{val}"]
      tmp['end'] = params["end#{val}"]
      tmp['time'] = params["time#{val}"]
      tmp['time_to'] = params["time_to#{val}"]
      ary["obj#{val}"] = tmp
    end
    ary
  end
```

## 6. controllerでタイムエントリーを作っていく

下記で行っている処理の流れは次のとおりです。
1. アクセストークンを取得する
1. viewからユーザの入力データを取得する
1. 期間入力は、末尾にインデックスをつけているので、実装した分をseparateで取得する
1. 期間・時間が全て入力されているかをチェックする(見ての通り、validateが不十分です)
1. 期間・時間を秒単位に変換する
1. タイムエントリーを作成する
1. 作成したタイムエントリーからIDを取得する。
1. 取得した日時は日本標準時になっているので、世界標準時に変換する
1. タイムエントリーを世界標準時で更新する

何が起こっても想定以上の登録ができないように、1期間あたり6タイムエントリーを作ると処理を抜けます。

```ruby:tasks_controller
  def gen
    if init
      title = params['title']
      team_id = params['team_id']
      key = params['key']
      url = params['url']

      cls = separate(params)
      cls.each do |_key_value, val|
        index = 0
        object = val
        next if object['begin'].blank?
        next if object['end'].blank?
        next if object['time'].blank?
        next if object['time_to'].blank?

        # 時間変換
        beg_date = DateTime.parse(object['begin'] + ' ' + object['time'])
        end_date = DateTime.parse(object['end'] + ' ' + object['time'])
        time = DateTime.parse(object['end'] + ' ' + object['time_to']).to_i - end_date.to_i
        time = (time / 60) / 60 # 時間経過

        loop do
          tmp_date = beg_date + index
          # エンティティ作成
          response = @access_token.post('/api/v1/time_entries',
                                        body: {
                                          task: { title: title,
                                                  team_id: team_id,
                                                  key: key,
                                                  url: url },
                                          parent: { title: '会社',
                                                    key: '会社',
                                                    url: '' }
                                        })

          trace_id = params['id']
          record_id = response.parsed['id']

          # 日本標準時を逆算
          tmp_date_uc = tmp_date - (Rational(1, 24) * 9)
          stop_date = tmp_date_uc + (Rational(1, 24) * time)

          # エンティティ更新
          @res = @access_token.patch("/api/v1/time_entries/#{record_id}",
                                     body: {
                                       time_entry: { started_at: tmp_date_uc.to_i,
                                                     stopped_at: stop_date.to_i,
                                                     time_trackable_id: trace_id }
                                     })

          index += 1
          break if end_date == tmp_date || index > 6
        end
      end
    end
  end
```

# できたこと
毎日ぽちぽちいれていた朝礼が一瞬で入力できるようになりました。