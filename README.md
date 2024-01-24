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

2. PHPのRedis用の拡張， phpredis を導入<br>
vim Dockerfileに追記↓
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
1. 会員情報を保存するテーブルをデータベースに作成
```
CREATE TABLE `users` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    `name` TEXT NOT NULL,
    `email` TEXT NOT NULL,
    `password` TEXT NOT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

2. 会員登録フォームの作成（暗号化も行う）<br>
vim public/signup.php↓
```
<?php
// DBに接続
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');

if (!empty($_POST['name']) && !empty($_POST['email']) && !empty($_POST['password'])) {
  // POSTで name と email と password が送られてきた場合はDBへの登録処理をする

  // ソルトを決める(ランダム)
  $salt = bin2hex(random_bytes(32));

  // ストレッチング
  $password_hash = $_POST['password'];
  foreach (range(1, 100) as $count) {
    // ソルトを追加してハッシュ化... を100回繰り返す
    $password_hash = hash('sha256', $password_hash . $salt);
  }

  // insertする
  $insert_sth = $dbh->prepare("INSERT INTO users (name, email, password) VALUES (:name, :email, :password)");
  $insert_sth->execute([
    ':name' => $_POST['name'],
    ':email' => $_POST['email'],
    ':password' => hash('sha256', $_POST['password'] . $salt) . $salt,
  ]);

  // 処理が終わったら完了画面にリダイレクト
  header("HTTP/1.1 302 Found");
  header("Location: /signup_finish.php");
  return;
}
?>

<h1>会員登録</h1>

<!-- 登録フォーム -->
<form method="POST">
  <!-- input要素のtype属性は全部textでも動くが、適切なものに設定すると利用者は使いやすい -->
  <label>
    名前:
    <input type="text" name="name">
  </label>
  <br>
  <label>
    メールアドレス:
    <input type="email" name="email">
  </label>
  <br>
  <label>
    パスワード:
    <input type="password" name="password" min="6" autocomplete="new-password">
  </label>
  <br>
  <button type="submit">決定</button>
</form>
```

vim public/signup_finish.php↓
```
<h1>会員登録完了</h1>

会員登録が完了しました。
```

3. ログインフォームの作成<br>
vim public/login.php↓
```
<?php
// DBに接続
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');

if (!empty($_POST['email']) && !empty($_POST['password'])) {
  // POSTで email と password が送られてきた場合のみログイン処理をする

  // email から会員情報を引く
  $select_sth = $dbh->prepare("SELECT * FROM users WHERE email = :email ORDER BY id DESC LIMIT 1");
  $select_sth->execute([
    ':email' => $_POST['email'],
  ]);
  $user = $select_sth->fetch();

  if (empty($user)) {
    // 入力されたメールアドレスに該当する会員が見つからなければ、処理を中断しエラー用クエリパラメータ付きのログイン画面URLにリダイレクト
    header("HTTP/1.1 302 Found");
    header("Location: ./login.php?error=1");
    return;
  }

  // ハッシュ部分とソルトを分離
  $password_hash = mb_substr($user['password'], 0, 64); // 0文字目から64文字分がハッシュ
  $salt = mb_substr($user['password'], 64, 64); // 64文字目から64文字分がソルト


  // ストレッチング
  $input_password_hash = $_POST['password'];
  foreach (range(1, 100) as $count) {
    // ソルトを追加してハッシュ化... を100回繰り返す
    $input_password_hash = hash('sha256', $input_password_hash . $salt);
  }

  $correct_password = $input_password_hash === $password_hash;

  if (!$correct_password) {
    // パスワードが間違っていれば、処理を中断しエラー用クエリパラメータ付きのログイン画面URLにリダイレクト
    header("HTTP/1.1 302 Found");
    header("Location: ./login.php?error=1");
    return;
  }


  # ここからセッションの独自実装 (詳細は後期第1回授業参照) ##############
  // セッションIDの取得(なければ新規で作成&設定)
  $session_cookie_name = 'session_id';
  $session_id = $_COOKIE[$session_cookie_name] ?? base64_encode(random_bytes(64));
  if (!isset($_COOKIE[$session_cookie_name])) {
    setcookie($session_cookie_name, $session_id);
  }

  // 接続 (redisコンテナの6379番ポートに接続)
  $redis = new Redis();
  $redis->connect('redis', 6379);

  // Redisにセッション変数を保存しておくキー
  $redis_session_key = "session-" . $session_id;

  // Redisからセッションのデータを読み込み
  // 既にセッション変数(の配列)が何かしら格納されていればそれを，なければ空の配列を $session_values変数に保存
  $session_values = $redis->exists($redis_session_key)
    ? json_decode($redis->get($redis_session_key), true)
    : [];

  // セッションにログインできた会員情報の主キー(id)を設定
  $session_values["login_user_id"] = $user['id'];

  // セッションをRedisに保存
  $redis->set($redis_session_key, json_encode($session_values));
  # セッションここまで ###################################################


  // ログインが成功したらログイン完了画面にリダイレクト
  header("HTTP/1.1 302 Found");
  header("Location: ./login_finish.php");
  return;
}
?>

<h1>ログイン</h1>

<!-- ログインフォーム -->
<form method="POST">
  <!-- input要素のtype属性は全部textでも動くが、適切なものに設定すると利用者は使いやすい -->
  <label>
    メールアドレス:
    <input type="email" name="email">
  </label>
  <br>
  <label>
    パスワード:
    <input type="password" name="password" min="6" autocomplete="new-password">
  </label>
  <br>
  <button type="submit">決定</button>
</form>

<?php if(!empty($_GET['error'])): // エラー用のクエリパラメータがある場合はエラーメッセージ表示 ?>
<div style="color: red;">
  メールアドレスかパスワードが間違っています。
</div>
<?php endif; ?>
```

vim public/login_finish.php↓
```
<?php
// セッションIDの取得(なければ新規で作成&設定)
$session_cookie_name = 'session_id';
$session_id = $_COOKIE[$session_cookie_name] ?? base64_encode(random_bytes(64));
if (!isset($_COOKIE[$session_cookie_name])) {
    setcookie($session_cookie_name, $session_id);
}

// 接続 (redisコンテナの6379番ポートに接続)
$redis = new Redis();
$redis->connect('redis', 6379);

// Redisにセッション変数を保存しておくキー
$redis_session_key = "session-" . $session_id; 

// Redisからセッションのデータを読み込み
// 既にセッション変数(の配列)が何かしら格納されていればそれを，なければ空の配列を $session_values変数に保存
$session_values = $redis->exists($redis_session_key)
  ? json_decode($redis->get($redis_session_key), true) 
  : []; 

// セッションにログインIDが無ければ (=ログインされていない状態であれば) ログイン画面にリダイレクトさせる
if (empty($session_values['login_user_id'])) {
  header("HTTP/1.1 302 Found");
  header("Location: ./login.php");
  return;
}


// DBに接続
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');
// セッションにあるログインIDから、ログインしている対象の会員情報を引く
$insert_sth = $dbh->prepare("SELECT * FROM users WHERE id = :id");
$insert_sth->execute([
    ':id' => $session_values['login_user_id'],
]);
$user = $insert_sth->fetch();
?>

<h1>ログイン完了</h1>

<p>
  ログイン完了しました!
</p>
<hr>
<p>
  また、あなたが現在ログインしている会員情報は以下のとおりです。
</p>
<dl> <!-- 登録情報を出力する際はXSS防止のため htmlspecialchars() を必ず使いましょう -->
  <dt>ID</dt>
  <dd><?= htmlspecialchars($user['id']) ?></dd>
  <dt>メールアドレス</dt>
  <dd><?= htmlspecialchars($user['email']) ?></dd>
  <dt>名前</dt>
  <dd><?= htmlspecialchars($user['name']) ?></dd>
</dl>
```


PHPのセッション機能を使う
1. php.iniを整える<br>
vim php.ini↓
```
post_max_size = 5M
upload_max_filesize = 5M
```
vim Dockerfile↓
```diff
RUN install -o www-data -g www-data -d /var/www/upload/image/

- RUN echo -e "post_max_size = 5M\nupload_max_filesize = 5M" >> ${PHP_INI_DIR}/php.ini
+ COPY ./php.ini ${PHP_INI_DIR}/php.ini
```
2. 既存コードの書き換え
vim public/login.php↓
```diff
- 
-   # ここからセッションの独自実装 (詳細は後期第1回授業参照) ##############
-   // セッションIDの取得(なければ新規で作成&設定)
-   $session_cookie_name = 'session_id';
-   $session_id = $_COOKIE[$session_cookie_name] ?? base64_encode(random_bytes(64));
-   if (!isset($_COOKIE[$session_cookie_name])) {
-     setcookie($session_cookie_name, $session_id);
-   }
- 
-   // 接続 (redisコンテナの6379番ポートに接続)
-   $redis = new Redis();
-   $redis->connect('redis', 6379);
- 
-   // Redisにセッション変数を保存しておくキー
-   $redis_session_key = "session-" . $session_id;
- 
-   // Redisからセッションのデータを読み込み
-   // 既にセッション変数(の配列)が何かしら格納されていればそれを，なければ空の配列を $session_values変数に保存
-   $session_values = $redis->exists($redis_session_key)
-     ? json_decode($redis->get($redis_session_key), true)
-     : [];
+   session_start();

  // セッションにログインできた会員情報の主キー(id)を設定
-   $session_values["login_user_id"] = $user['id'];
- 
-   // セッションをRedisに保存
-   $redis->set($redis_session_key, json_encode($session_values));
-   # セッションここまで ###################################################
- 
+   $_SESSION["login_user_id"] = $user['id'];

  // ログインが成功したらログイン完了画面にリダイレクト
  header("HTTP/1.1 302 Found");
```

vim public/login_finish.php↓
```diff
<?php
// セッションIDの取得(なければ新規で作成&設定)
- $session_cookie_name = 'session_id';
- $session_id = $_COOKIE[$session_cookie_name] ?? base64_encode(random_bytes(64));
- if (!isset($_COOKIE[$session_cookie_name])) {
-     setcookie($session_cookie_name, $session_id);
- }

- // 接続 (redisコンテナの6379番ポートに接続)
- $redis = new Redis();
- $redis->connect('redis', 6379);

- // Redisにセッション変数を保存しておくキー
- $redis_session_key = "session-" . $session_id; 

- // Redisからセッションのデータを読み込み
- // 既にセッション変数(の配列)が何かしら格納されていればそれを，なければ空の配列を $session_values変数に保存
- $session_values = $redis->exists($redis_session_key)
-   ? json_decode($redis->get($redis_session_key), true) 
-   : []; 
+ session_start();

// セッションにログインIDが無ければ (=ログインされていない状態であれば) ログイン画面にリダイレクトさせる
- if (empty($session_values['login_user_id'])) {
+ if (empty($_SESSION['login_user_id'])) {
  header("HTTP/1.1 302 Found");
  header("Location: ./login.php");
  return;
}
// DBに接続
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');
// セッションにあるログインIDから、ログインしている対象の会員情報を引く
$insert_sth = $dbh->prepare("SELECT * FROM users WHERE id = :id");
$insert_sth->execute([
-     ':id' => $session_values['login_user_id'],
+     ':id' => $_SESSION['login_user_id'],
]);
$user = $insert_sth->fetch();
?>
```
PHPパスワード暗号化機能を使う
vim public/login.php↓
```diff
    return;
  }

-   // ハッシュ部分とソルトを分離
-   $password_hash = mb_substr($user['password'], 0, 64); // 0文字目から64文字分がハッシュ
-   $salt = mb_substr($user['password'], 64, 64); // 64文字目から64文字分がソルト

-   // ストレッチング
-   $input_password_hash = $_POST['password'];
-   foreach (range(1, 100) as $count) {
-     // ソルトを追加してハッシュ化... を100回繰り返す
-     $input_password_hash = hash('sha256', $input_password_hash . $salt);
-   }
- 
-   $correct_password = $input_password_hash === $password_hash;
+   $correct_password = password_verify($_POST['password'], $user['password']);

  if (!$correct_password) {
    // パスワードが間違っていれば、処理を中断しエラー用クエリパラメータ付きのログイン画面URLにリダイレクト
```
vim public/signup.php↓
```diff
if (!empty($_POST['name']) && !empty($_POST['email']) && !empty($_POST['password'])) {
  // POSTで name と email と password が送られてきた場合はDBへの登録処理をする

-   // ソルトを決める(ランダム)
-   $salt = bin2hex(random_bytes(32));
- 
-   // ストレッチング
-   $password_hash = $_POST['password'];
-   foreach (range(1, 100) as $count) {
-     // ソルトを追加してハッシュ化... を100回繰り返す
-     $password_hash = hash('sha256', $password_hash . $salt);
-   }
- 
  // insertする
  $insert_sth = $dbh->prepare("INSERT INTO users (name, email, password) VALUES (:name, :email, :password)");
  $insert_sth->execute([
    ':name' => $_POST['name'],
    ':email' => $_POST['email'],
-     ':password' => hash('sha256', $_POST['password'] . $salt) . $salt,
+     ':password' => password_hash($_POST['password'], PASSWORD_DEFAULT),
  ]);

  // 処理が終わったら完了画面にリダイレクト
```

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
