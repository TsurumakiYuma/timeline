Reddisのインストール
1. Redisサーバーをインストール<br>
vim compose.ymlに追記↓
```diff
    command: >
      mysqld
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci
      --max_allowed_packet=4MB
+  redis:
+    container_name: redis
+    image: redis:latest
+    ports:
+      - 6379:6379
volumes:
  image:
    driver: local
  mysql:
```

2. PHPのRedis用の拡張， phpredis を導入します
vim Dockerfile↓
```diff
FROM php:8.1-fpm-alpine AS php

RUN docker-php-ext-install pdo_mysql

+ RUN apk add git
+ RUN git clone https://github.com/phpredis/phpredis.git /usr/src/php/ext/redis
+ RUN docker-php-ext-install redis

RUN install -o www-data -g www-data -d /var/www/upload/image/

RUN echo -e "post_max_size = 5M\nupload_max_filesize = 5M" >> ${PHP_INI_DIR}/php.ini
```

会員登録
1. 会員情報を保存するテーブルをデータベースに作成する
```
CREATE TABLE `users` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    `name` TEXT NOT NULL,
    `email` TEXT NOT NULL,
    `password` TEXT NOT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

2. 会員登録フォームの作成（メアドやハッシュ化も行う）
vim public/signup.php
vim public/signup_finish.php

3. ログインフォームの作成
vim public/login.php
vim public/login_finish.php

PHPのセッション機能を使う
1php.iniの追加

設定画面の追加
1まずアイコン画像のファイル名を保持するカラムを会員情報テーブル users に追加します。
ALTER TABLE `users` ADD COLUMN icon_filename TEXT DEFAULT NULL;
ALTER TABLE `users` ADD COLUMN introduction TEXT DEFAULT NULL;
ALTER TABLE `users` ADD COLUMN cover_filename TEXT DEFAULT NULL;
ALTER TABLE `users` ADD COLUMN birthday DATE DEFAULT NULL;

2設定画面の作成
mkdir public/setting
vim public/setting/index.php
vim public/setting/icon.php
vim public/setting/introduction.php
vim public/setting/cover.php
vim public/setting/birthday.php

3プロフィールページの作成
vim public/profile.php


会員サービスに紐づけた掲示板を作る
1新しく掲示板投稿テーブルを作成します。
CREATE TABLE `bbs_entries` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    `user_id` INT UNSIGNED NOT NULL,
    `body` TEXT NOT NULL,
    `image_filename` TEXT DEFAULT NULL,

vim public/bbs.php

ログインや会員登録フォームまわりの導線を整える


フォロー機能を作ってみましょう
CREATE TABLE `user_relationships` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    `followee_user_id` INT UNSIGNED NOT NULL,
    `follower_user_id` INT UNSIGNED NOT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
);

vim public/follow.php

vim public/follow_list.php 

vim public/follower_list.php 


タイムラインを作ってみましょう
3パターンある
vim timeline_in.php
subquery


会員一覧画面を作る
vim public/users.php

タイムラインをJSでレンダリングする
vim public/timeline_json.php
