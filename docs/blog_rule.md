FREEPLUS開発チーム橋本です。

今回は新たに技術ブログを始めるにあたって社内で決めたこと、ルール化したことを書いていきたいと思います。  


## 1. 記事執筆前  

#### 1. チームで打ち合わせして決める  
まず、朝会または週一で実施するチームミーティング内で何を書くか決めるパターン。  
次回のブログの担当者とネタ出しが目的。  
チームで内容を決めておくことで、担当者が記事を書くハードルが下げることができるかと考えています。  

#### 2. 書きたいものができたらすぐに書く  
**思いついたら書くパターン**  
とくに制限なく書けるルールにしています。  

編集会議で決めて〜という流れだけですと、時間がかかり投稿数が伸びないことが予想されるとともに、  
書かされている感がでてしまうのではないかという危惧から「いつ誰が書いても良い」としました。  

## 2. 記事の書き方  
チームメンバ各自が好きなMarkDown Editorを使用してMarkDownで執筆してもらっています。  

ネットで調べてみるとATOM、Visual Studio Codeあたりがおすすめだそうです。  
http://uxmilk.jp/43199  

ちなみに私は、RubyMineに「Gfm」というプラグインを入れて編集・プレビューをしています。  

## 3. ブログガイドライン  
ブログの内容によって、企業にとっての機密情報や個人情報の漏えいの危険性、  
また、それによって企業イメージを損なうといった問題が起きないよう、一定の配慮が必要になります。  
(良かれと思ってやったのに、炎上という自体になっては目も当てられません)

そのため、一般的には「ブログガイドライン」というものが企業によって制定されるようです。  
例として以下の情報を参考にしています。  

- IBMのブログガイドライン  
http://d.hatena.ne.jp/mintblue_erica/20050812/p1  

- 日立ソリューションズ  
社員ブログと企業のガイドラインについて  
http://securityblog.jp/monthly/298.html  

**※ FREEPLUS独自のブログガイドライン**   
どういう形でもかまわないので、執筆者の名前の記載をお願いしています。  
あとは上記の内容に反していなければ問題ないと思います。    

具体的には、、、そのうち文章として作成できれば良いなと思っています。  

## 4. 自動校正  
各自作成した原稿はGithubで管理をしています。  
リポジトリにPushすると自動的にTravisCIで校正ツールによるテストが実行されるようになっています。  

このあたりの仕組みは、以下のページを参考に作成させて頂きました。

- Qiita - textlintをTravis CIで動かして継続的に文章をチェックする
https://qiita.com/azu/items/e36501d25593d008f6ac  

- Web Scratch - JavaScriptでルールを書けるテキスト/Markdownの校正ツール textlint を作った
http://efcl.info/2014/12/30/textlint/  

#### 1. 使用ツール  
textlint  
http://efcl.info/2015/09/10/introduce-textlint/  

#### 2. フォルダ構成  
  ````
  .textlintrc  
  .travis.yml  
  README.md  
  docs/  
    └作成した記事  
  old/  
    └過去記事、不要なファイル  
  package.json  
  ````
  docsフォルダ内の「.md」ファイルのみを対象にしています。  
  プロジェクト全体をテストはしません。  
  また、念のため、過去の記事退避用に以下のoldフォルダも用意しています。  

#### 3. テストルール  
現在ツールで使用しているルールは以下になります。  
使えそうなものが他にもあったら追加大歓迎です。  

````
# 英語のよくあるスペルミスをチェックする  
"textlint-rule-common-misspellings": "^1.0.1",  
#
# 一文に利用できる、の数をチェックするルール  
"textlint-rule-max-ten": "^2.0.3",  
#
# 敬体(ですます調)と常体(である調)の表記を統一する  
"textlint-rule-no-mix-dearu-desumasu": "^3.0.3",  
#
# 日本語関係のルールセット  
"textlint-rule-preset-japanese": "^1.3.4",  
#
# WEB+DB PRESS用語統一ルールをベースにしたazu/technical-word-rulesの辞書で単語チェック  
"textlint-rule-spellcheck-tech-word": "^5.0.0"  
````

使用しているルールは以下のファイルに記載されています。  

package.json  
````
{
  "name": "textlint_test",  
  "version": "1.0.0",  
  "description": "",  
  "scripts": {  
    "textlint": "textlint -f pretty-error docs/"  
  },  
  "author": "",  
  "license": "ISC",  
  "devDependencies": {  
    "textlint": "^8.2.1",  
    "textlint-rule-common-misspellings": "^1.0.1",  
    "textlint-rule-max-ten": "^2.0.3",  
    "textlint-rule-no-mix-dearu-desumasu": "^3.0.3",  
    "textlint-rule-preset-japanese": "^1.3.4",  
    "textlint-rule-spellcheck-tech-word": "^5.0.0"  
  }  
}  
````

自動テストの設定は以下になります。  

.travis.yml  
````
sudo: false  
language: node_js  
node_js: "stable"  
script:  
- npm run textlint  
````


**※ FREEPLUS独自のルール**    
FREEPLUSの入力方法のルールをlintルールにしてみたい。。。ですが、できればやります。  

## 5. レビュー  
テストが成功したら、通常のソースコードと同様に他者によるレビューを実施をおねがいしています。    
ただし、レビュー実施者は特に定めません。  

**レビュー観点は、上記ガイドラインの項目で上げた内容について問題がなければ良いと思います。**  

レビューを実施し問題がなければ、masterにマージして完了になります。  


## 6. 公開  
各自、はてなブログのダッシュボードから記事を新規作成してください。  
ブログの管理ページは以下になります。  
http://blog.hatena.ne.jp/  

**公開はレビューさえ通ればいつでもOK！！スピード重視！**  

## 以上。  
このような形で運用を開始しました。  
今後も運用ルールを見直しつつより良い記事を書けるよう努力していきます。  

また、MarkDownの自動校正ツールはなかなかおもしろかったので、  
別途詳しく記事にできたらなぁ・。。と思っています。

**ツッコミやアドバイスお待ちしております。**  



