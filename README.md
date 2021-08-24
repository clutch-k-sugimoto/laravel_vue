# nginx + Laravel8 + Vue3 + TypeScript (+ vuetify)の環境構築手順のあれこれ  
## 概要
これできたらクールと巷で噂のモダンなフロントエンドフレームワーク筆頭である`nginx + Laravel + vue`のベースとなるdocker構成。  
備忘も兼ねて、内容記述と構築手順を記す。  
<br>

## ベース構成  
OSはすべてAlpineLinux （軽量化というクールな選択に乗っかるスタイル）  
* WEB  
  nginx 1.18  

* APP  
  PHP 8.0.9 + fpm   
  composer 2.1.5  
  node 16.7.0  

* DB  
  mariadb 10.6.4  
<br>

## 導入方法  
### サーバを立てる  
0. `docker-compose.yaml`にある、各コンテナ名（`container_name`）は、好きな名前にしておくと良いかも。  
    ```
    version: '3.8'

    services:
    app:
        container_name: test_app  <-- こいつ
        build:
        context: .
        dockerfile: ./docker/php/Dockerfile
    ```

1. `.env`を作り、接続情報やらポートやらを以下のように書いて、`docker-compose.yaml`の穴埋めをする。  
    例）
    ```
    WEB_PORT=8080               # ブラウザから叩いたときの接続ポート番号 他のサービスと同時に動かしてるときなどは変更がいる
    DB_PORT=3306                # DBサーバのポート番号 WEB_PORT同様、他とかぶってたら適宜変えること

    DB_NAME=test_db             # 接続先（コンテナ名）
    DB_USER=test_user           # ユーザ名
    DB_PASSWORD=test_password   # パスワード
    DB_ROOT_PASSWORD=root       # rootで入ったときのパスワード
    ```

2. コンテナのビルドと起動  
    appにマルチステージビルドを採用しているため、単純に`up`ではなく明確にビルドのオプションを付ける必要があるらし。
    ```
    docker-compose up --build -d
    ```
<br>

### app設定  
これ以降は各個人のプロジェクトの作成の解説となる。  
割とやること多いので覚悟すべし。
<br>
1. 必要なものがあるか確認する  
    とりあえず、`composer`,`node`,`npm`が生存していることを確認する。  

    composer
    ```
    % docker exec -it test_app composer -V
    Composer version 2.1.5 2021-07-23 10:35:47
    ```

    node
    ```
    % docker exec -it test_app node -v
    v16.7.0
    ```

    npm
    ```
    % docker exec -it test_app npm -v
    7.20.3
    ```

2. Laravelのプロジェクト作成  
    `laravel 8系の最新`を指定してプロジェクトを作成する。  
    固定する必要があるなら、`*`を実数にしてやれば良い。  

    ホストのsrcでマウントしているため、ざっとローカルにファイルが増える。  
    出来合いのプロジェクトから持ってくる場合は特に以降の作業は必要ないかも。  

    これで入って
    ```
    docker exec -it test_app ash
    ```

    こいつを実行
    ```
    composer create-project --prefer-dist "laravel/laravel=8.*" .
    ```

    インストールしたあとに、これやってみて同じようなものが出ればおｋ  
    同時にUsageが大量に出るが、一番上にちょろっと出てる
    ```
    % php artisan -v
    Laravel Framework 8.55.0
    ```
3. migrateを実行して、とりあえずDBの接続がOKか確認する  
   デフォルトで入っているmigrationsを使って、`docker-compose.yaml`で書いたターゲットに登録されるか見る  
   以下のような結果になったらおｋ、エラーになったら接続情報が間違ってる可能性
    ```
    % php artisan migrate
    Migration table created successfully.
    Migrating: 2014_10_12_000000_create_users_table
    Migrated:  2014_10_12_000000_create_users_table (36.26ms)
    Migrating: 2014_10_12_100000_create_password_resets_table
    Migrated:  2014_10_12_100000_create_password_resets_table (30.10ms)
    Migrating: 2019_08_19_000000_create_failed_jobs_table
    Migrated:  2019_08_19_000000_create_failed_jobs_table (28.98ms)
    Migrating: 2019_12_14_000001_create_personal_access_tokens_table
    Migrated:  2019_12_14_000001_create_personal_access_tokens_table (46.82ms)
    ```
4. Laravel-UIのインストール  
   モダンなフロントエンドフレームワークを使う意思表示をLaravelに対して行う  
    ```
    composer require laravel/ui
    ```

5. Vue本体並びにその他関連した必要なモノをpackage.jsonに追加する  
   じゃぁ何使うねんということで、Vueつかうねんと
    ```
    php artisan ui vue
    ```

6. Vue3に変更する  
    デフォルトで入れてしまうと`2.x`となってしまうため、以下で排除する
    ```
    npm uninstall vue
    npm uninstall vue-loader
    npm uninstall vue-template-compiler
    ```

    排除したもののVer3とその他諸々を改めて入れる
    ```
    npm install -D vue@next
    npm install -D @vue/compiler-sfc
    npm install -D vue-style-loader
    npm install -D vue-loader
    npm install -D vue-router
    ```

7. これはおまけでvuetifyを入れる （飛ばしてもおｋ）  
    vue3に未対応であるためAlpha版を入れる  
    ```
    npm install -g @vue/cli
    vue add vuetify

    # 選択問題が出たら以下を選ぶ
    ? Choose a preset: (Use arrow keys)
    Default (recommended)
    Preview (Vuetify 3 + Vite)
    Prototype (rapid development)
    V3 (alpha)                      <----- こいつ
    Configure (advanced)
    ```

    念の為一発叩いとく（）  
    結果が以下なら特に問題なし
    ```
    # npm install
    up to date, audited 850 packages in 1s
    ```

8. ビルド  
   webpack.mix.jsが以下のようになるように修正
   ```
    mix.js('resources/js/app.js', 'public/js')
    .vue()
    .sass('resources/sass/app.scss', 'public/css');

    // ここから下はvuetifyを入れたら自分で追加
    mix.options({
        legacyNodePolyfills: false
    });

   ```

   ビルド実行
   ```
   npm run dev
   ```

   こんなのがでたらおｋ
   ```
   ✔ Compiled Successfully in 16525ms
    ┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┬──────────┐
    │                                                                                                                                     File │ Size     │
    ├──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────┤
    │                                                                                                                               /js/app.js │ 1.99 MiB │
    │                                                                                                                              css/app.css │ 178 KiB  │
    └──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┴──────────┘
    webpack compiled successfully
   ```
<br>

## 参考文献  
* 【前編】絶対に失敗しないDockerでLaravel + Vue.jsの開発環境（LEMP環境）を構築する方法〜MacOS Intel Chip対応〜  
https://yutaro-blog.net/2021/04/29/docker-laravel-vuejs-2/

* 【後編】絶対に失敗しないDockerでLaravel + Vue.jsの開発環境（LEMP環境）を構築する方法〜MacOS Intel Chip対応〜  
https://yutaro-blog.net/2021/04/30/docker-laravel-vuejs-3/

* Vue.js + LaravelでシンプルなSPA構築チュートリアル：概要編  
https://qiita.com/minato-naka/items/2d2def4d66ec88dc3ca2

* Laravel8でVue 3を使う  
https://reffect.co.jp/laravel/laravel8-vue3

* Vuetify 3 Alpha  
https://next.vuetifyjs.com/ja/getting-started/installation/

* はじめてのTypeScript入門  
https://reffect.co.jp/html/hello-typescript-tutorial
