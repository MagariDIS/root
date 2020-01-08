<p>InHouseのNode-REDから外部入力を受け付けるためTwitter APIを使ってみた<br/>
APIを利用できる状態(初期設定)がすごく大変でした。APIを利用するためには、アプリケーションの情報(Key,ID)を事前に取得して必要があります。
実際やってみたときは、「どこからAPIのID取得するの？ってなって１０分くらいさまよいました。。。」ということで早速やってみたいと思います。</p>

<h3>前提条件</h3>
<ul>
<li>ツイッターアカウントを取得済みで、電話番号が連携されていること。</li>
<li>InHouseにNode-REDの実行環境が構築済みであること</li>
</ul>

<h3>事前作業</h3>

<p>Twitter APIを実行するには、こちらの４つの情報が必要になります。</p>
<ol>
<li>consumer_key　　　・・・アプリケーションのキー</li>
<li>consumer_secret　　・・・アプリケーションのシークレット</li>
<li>access_token_key　　・・・ツイッターアカウントのキー</li>
<li>access_token_secret　・・・ツイッターアカウントのシークレット</li>
</ol>
<h4>アプリケーションを登録してAPIの取得</h4>
<p>まずは、consumer_keyとconsumer_secretを取得します。</p>
<p>１．下記URLよりアプリケーションを登録をおこないます。</p>
<p><a href="https://apps.twitter.com/app/new">https://apps.twitter.com/app/new</a></p>
<p>以下のように必要な情報入力していきます。</p>
<p>※ Nameは一意の名称である必要があるので、他の人が既に使用している名前での登録はできません。</p>
<p>※ Descriptionは１０文字以上で無いとエラーになるので、最低１０文字は入れてください。</p>

![](/images/nodered-ipconfig.png)



Node-REDのフロー内で値を共有できる変数をcontext.global.xxxで作成する事ができます。
