**Reddisのインストール**
1. Redisサーバーをインストール<br>
vim compose.ymlに追記
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

**会員登録**
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

**PHPのセッション機能**
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

2. 既存コードの書き換え<br>
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

3. PHPパスワード暗号化機能を使う<br>
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

**設定画面の追加**
1. まずカラムを会員情報テーブル users に追加
```
ALTER TABLE `users` ADD COLUMN icon_filename TEXT DEFAULT NULL;
ALTER TABLE `users` ADD COLUMN introduction TEXT DEFAULT NULL;
ALTER TABLE `users` ADD COLUMN cover_filename TEXT DEFAULT NULL;
ALTER TABLE `users` ADD COLUMN birthday DATE DEFAULT NULL;
```

2. 設定画面の作成<br>
mkdir public/setting<br>
vim public/setting/index.php
```
<?php
session_start();

if (empty($_SESSION['login_user_id'])) {
  header("HTTP/1.1 302 Found");
  header("Location: /login.php");
  return;
}

// DBに接続
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');
// セッションにあるログインIDから、ログインしている対象の会員情報を引く
$select_sth = $dbh->prepare("SELECT * FROM users WHERE id = :id");
$select_sth->execute([
    ':id' => $_SESSION['login_user_id'],
]);
$user = $select_sth->fetch();
?>

<a href="/bbs.php">掲示板に戻る</a>

<h1>設定画面</h1>

<p>
  現在の設定
</p>
<dl> <!-- 登録情報を出力する際はXSS防止のため htmlspecialchars() を必ず使いましょう -->
  <dt>ID</dt>
  <dd><?= htmlspecialchars($user['id']) ?></dd>
  <dt>メールアドレス</dt>
  <dd><?= htmlspecialchars($user['email']) ?></dd>
  <dt>名前</dt>
  <dd><?= htmlspecialchars($user['name']) ?></dd>
</dl>

<ul>
  <li><a href="./name.php">名前設定</a></li>
  <li><a href="./icon.php">アイコン設定</a></li>
  <li><a href="./cover.php">カバー画像設定</a></li>
  <li><a href="./birthday.php">生年月日設定</a></li>
  <li><a href="./introduction.php">自己紹介文設定</a></li>
</ul>
```

vim public/setting/icon.php↓
```
<?php
session_start();

if (empty($_SESSION['login_user_id'])) {
  header("HTTP/1.1 302 Found");
  header("Location: /login.php");
  return;
}

// DBに接続
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');
// セッションにあるログインIDから、ログインしている対象の会員情報を引く
$select_sth = $dbh->prepare("SELECT * FROM users WHERE id = :id");
$select_sth->execute([
    ':id' => $_SESSION['login_user_id'],
]);
$user = $select_sth->fetch();

if (isset($_POST['image_base64'])) {
  // POSTで送られてくるフォームパラメータ image_base64 がある場合

  $image_filename = null;
  if (!empty($_POST['image_base64'])) {
    // 先頭の data:~base64, のところは削る
    $base64 = preg_replace('/^data:.+base64,/', '', $_POST['image_base64']);

    // base64からバイナリにデコードする
    $image_binary = base64_decode($base64);

    // 新しいファイル名を決めてバイナリを出力する
    $image_filename = strval(time()) . bin2hex(random_bytes(25)) . '.png';
    $filepath =  '/var/www/upload/image/' . $image_filename;
    file_put_contents($filepath, $image_binary);
  }

  // ログインしている会員情報のnameカラムを更新する
  $update_sth = $dbh->prepare("UPDATE users SET icon_filename = :icon_filename WHERE id = :id");
  $update_sth->execute([
      ':id' => $user['id'],
      ':icon_filename' => $image_filename,
  ]);

  // 処理が終わったらリダイレクトする
  // リダイレクトしないと，リロード時にまた同じ内容でPOSTすることになる
  header("HTTP/1.1 302 Found");
  header("Location: ./icon.php");
  return;
}

?>
<a href="./index.php">設定一覧に戻る</a>

<h1>アイコン画像設定/変更</h1>

<div>
  <?php if(empty($user['icon_filename'])): ?>
  現在未設定
  <?php else: ?>
  <img src="/image/<?= $user['icon_filename'] ?>"
    style="height: 5em; width: 5em; border-radius: 50%; object-fit: cover;">
  <?php endif; ?>
</div>

<form method="POST">
  <div style="margin: 1em 0;">
    <input type="file" accept="image/*" name="image" id="imageInput">
  </div>
  <input id="imageBase64Input" type="hidden" name="image_base64"><!-- base64を送る用のinput (非表示) -->
  <canvas id="imageCanvas" style="display: none;"></canvas><!-- 画像縮小に使うcanvas (非表示) -->
  <button type="submit">アップロード</button>
</form>

<hr>

<script>
document.addEventListener("DOMContentLoaded", () => {
  const imageInput = document.getElementById("imageInput");
  imageInput.addEventListener("change", () => {
    if (imageInput.files.length < 1) {
      // 未選択の場合
      return;
    }

    const file = imageInput.files[0];
    if (!file.type.startsWith('image/')){ // 画像でなければスキップ
      return;
    }

    // 画像縮小処理
    const imageBase64Input = document.getElementById("imageBase64Input"); // base64を送るようのinput
    const canvas = document.getElementById("imageCanvas"); // 描画するcanvas
    const reader = new FileReader();
    const image = new Image();
    reader.onload = () => { // ファイルの読み込み完了したら動く処理を指定
      image.onload = () => { // 画像として読み込み完了したら動く処理を指定

        // 元の縦横比を保ったまま縮小するサイズを決めてcanvasの縦横に指定する
        const originalWidth = image.naturalWidth; // 元画像の横幅
        const originalHeight = image.naturalHeight; // 元画像の高さ
        const maxLength = 1000; // 横幅も高さも1000以下に縮小するものとする
        if (originalWidth <= maxLength && originalHeight <= maxLength) { // どちらもmaxLength以下の場合そのまま
            canvas.width = originalWidth;
            canvas.height = originalHeight;
        } else if (originalWidth > originalHeight) { // 横長画像の場合
            canvas.width = maxLength;
            canvas.height = maxLength * originalHeight / originalWidth;
        } else { // 縦長画像の場合
            canvas.width = maxLength * originalWidth / originalHeight;
            canvas.height = maxLength;
        }

        // canvasに実際に画像を描画 (canvasはdisplay:noneで隠れているためわかりにくいが...)
        const context = canvas.getContext("2d");
        context.drawImage(image, 0, 0, canvas.width, canvas.height);

        // canvasの内容をbase64に変換しinputのvalueに設定
        imageBase64Input.value = canvas.toDataURL();
      };
      image.src = reader.result;
    };
    reader.readAsDataURL(file);
  });
});
</script>
```

vim public/setting/name.php
```
<?php
session_start();

// セッションにログインIDが無ければ (=ログインされていない状態であれば) ログイン画面にリダイレクトさせる
if (empty($_SESSION['login_user_id'])) {
  header("HTTP/1.1 302 Found");
  header("Location: /login.php");
  return;
}

// DBに接続
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');
// セッションにあるログインIDから、ログインしている対象の会員情報を引く
$insert_sth = $dbh->prepare("SELECT * FROM users WHERE id = :id");
$insert_sth->execute([
    ':id' => $_SESSION['login_user_id'],
]);
$user = $insert_sth->fetch();

if (isset($_POST['name'])) {
  // フォームから name が送信されてきた場合の処理

  // ログインしている会員情報のnameカラムを更新する
  $insert_sth = $dbh->prepare("UPDATE users SET name = :name WHERE id = :id");
  $insert_sth->execute([
      ':id' => $user['id'],
      ':name' => $_POST['name'],
  ]);
  // 成功したら成功したことを示すクエリパラメータつきのURLにリダイレクト
  header("HTTP/1.1 302 Found");
  header("Location: /setting/name.php?success=1");
  return;
}
?>
<a href="./index.php">設定一覧に戻る</a>

<h1>名前変更</h1>
<form method="POST">
  <input type="text" name="name" value="<?= htmlspecialchars($user['name']) ?>">
  <button type="submit">決定</button>
</form>

<?php if(!empty($_GET['success'])): ?>
<div>
  名前の変更処理が完了しました。
</div>
<?php endif; ?>
```

vim public/setting/introduction.php↓
```
<?php
session_start();

if (empty($_SESSION['login_user_id'])) {
  header("HTTP/1.1 302 Found");
  header("Location: /login.php");
  return;
}

// DBに接続
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');
// セッションにあるログインIDから、ログインしている対象の会員情報を引く
$select_sth = $dbh->prepare("SELECT * FROM users WHERE id = :id");
$select_sth->execute([
    ':id' => $_SESSION['login_user_id'],
]);
$user = $select_sth->fetch();

if (isset($_POST['introduction'])) {
  // フォームから introduction が送信されてきた場合の処理

  // ログインしている会員情報のintroductionカラムを更新する
  $update_sth = $dbh->prepare("UPDATE users SET introduction = :introduction WHERE id = :id");
  $update_sth->execute([
      ':id' => $user['id'],
      ':introduction' => $_POST['introduction'],
  ]);
  // 成功したら成功したことを示すクエリパラメータつきのURLにリダイレクト
  header("HTTP/1.1 302 Found");
  header("Location: ./introduction.php?success=1");
  return;
}
?>
<a href="./index.php">設定一覧に戻る</a>

<h1>自己紹介設定</h1>
<form method="POST">
  <textarea type="text" name="introduction" rows="5"
    ><?= htmlspecialchars($user['introduction'] ?? '') ?></textarea>
  <button type="submit">決定</button>
</form>

<?php if(!empty($_GET['success'])): ?>
<div>
  自己紹介文の設定処理が完了しました。
</div>
<?php endif; ?>
```

vim public/setting/cover.php
```
<?php
session_start();

if (empty($_SESSION['login_user_id'])) {
  header("HTTP/1.1 302 Found");
  header("Location: /login.php");
  return;
}

// DBに接続
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');
// セッションにあるログインIDから、ログインしている対象の会員情報を引く
$select_sth = $dbh->prepare("SELECT * FROM users WHERE id = :id");
$select_sth->execute([
    ':id' => $_SESSION['login_user_id'],
]);
$user = $select_sth->fetch();

if (isset($_POST['image_base64'])) {
  // POSTで送られてくるフォームパラメータ image_base64 がある場合

  $image_filename = null;
  if (!empty($_POST['image_base64'])) {
    // 先頭の data:~base64, のところは削る
    $base64 = preg_replace('/^data:.+base64,/', '', $_POST['image_base64']);

    // base64からバイナリにデコードする
    $image_binary = base64_decode($base64);

    // 新しいファイル名を決めてバイナリを出力する
    $image_filename = strval(time()) . bin2hex(random_bytes(25)) . '.png';
    $filepath =  '/var/www/upload/image/' . $image_filename;
    file_put_contents($filepath, $image_binary);
  }

  // ログインしている会員情報のカバー画像ファイル名を更新する
  $update_sth = $dbh->prepare("UPDATE users SET cover_filename = :cover_filename WHERE id = :id");
  $update_sth->execute([
      ':id' => $user['id'],
      ':cover_filename' => $image_filename,
  ]);

  // 処理が終わったらリダイレクトする
  // リダイレクトしないと，リロード時にまた同じ内容でPOSTすることになる
  header("HTTP/1.1 302 Found");
  header("Location: ./cover.php");
  return;
}

?>

<a href="./index.php">設定一覧に戻る</a>

<h1>カバー画像</h1>

<div>
  <?php if(empty($user['cover_filename'])): ?>
  現在未設定
  <?php else: ?>
  <img src="/image/<?= $user['cover_filename'] ?>"
    style="height: 5em; width: 10em; object-fit: cover;">
  <?php endif; ?>
</div>

<!-- フォームのPOST先はこのファイル自身にする -->
<form method="POST" action="./cover.php">
  <div style="margin: 1em 0;">
    <input type="file" accept="image/*" name="image" id="imageInput">
  </div>
  <input id="imageBase64Input" type="hidden" name="image_base64"><!-- base64を送る用のinput (非表示) -->
  <canvas id="imageCanvas" style="display: none;"></canvas><!-- 画像縮小に使うcanvas (非表示) -->
  <button type="submit">アップロード</button>
</form>

<hr>

<script>
document.addEventListener("DOMContentLoaded", () => {
  const imageInput = document.getElementById("imageInput");
  imageInput.addEventListener("change", () => {
    if (imageInput.files.length < 1) {
      // 未選択の場合
      return;
    }

    const file = imageInput.files[0];
    if (!file.type.startsWith('image/')){ // 画像でなければスキップ
      return;
    }

    // 画像縮小処理
    const imageBase64Input = document.getElementById("imageBase64Input"); // base64を送るようのinput
    const canvas = document.getElementById("imageCanvas"); // 描画するcanvas
    const reader = new FileReader();
    const image = new Image();
    reader.onload = () => { // ファイルの読み込み完了したら動く処理を指定
      image.onload = () => { // 画像として読み込み完了したら動く処理を指定

        // 元の縦横比を保ったまま縮小するサイズを決めてcanvasの縦横に指定する
        const originalWidth = image.naturalWidth; // 元画像の横幅
        const originalHeight = image.naturalHeight; // 元画像の高さ
        const maxLength = 1000; // 横幅も高さも1000以下に縮小するものとする
        if (originalWidth <= maxLength && originalHeight <= maxLength) { // どちらもmaxLength以下の場合そのまま
            canvas.width = originalWidth;
            canvas.height = originalHeight;
        } else if (originalWidth > originalHeight) { // 横長画像の場合
            canvas.width = maxLength;
            canvas.height = maxLength * originalHeight / originalWidth;
        } else { // 縦長画像の場合
            canvas.width = maxLength * originalWidth / originalHeight;
            canvas.height = maxLength;
        }

        // canvasに実際に画像を描画 (canvasはdisplay:noneで隠れているためわかりにくいが...)
        const context = canvas.getContext("2d");
        context.drawImage(image, 0, 0, canvas.width, canvas.height);

        // canvasの内容をbase64に変換しinputのvalueに設定
        imageBase64Input.value = canvas.toDataURL();
      };
      image.src = reader.result;
    };
    reader.readAsDataURL(file);
  });
});
</script>
```

vim public/setting/birthday.php
```
<?php
session_start();

if (empty($_SESSION['login_user_id'])) {
  header("HTTP/1.1 302 Found");
  header("Location: /login.php");
  return;
}

// DBに接続
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');
// セッションにあるログインIDから、ログインしている対象の会員情報を引く
$select_sth = $dbh->prepare("SELECT * FROM users WHERE id = :id");
$select_sth->execute([
    ':id' => $_SESSION['login_user_id'],
]);
$user = $select_sth->fetch();

if (isset($_POST['birthday'])) {
  // フォームから birthday が送信されてきた場合の処理

  // ログインしている会員情報のbirthdayカラムを更新する
  $update_sth = $dbh->prepare("UPDATE users SET birthday = :birthday WHERE id = :id");
  $update_sth->execute([
      ':id' => $user['id'],
      ':birthday' => $_POST['birthday'],
  ]);
  // 成功したら成功したことを示すクエリパラメータつきのURLにリダイレクト
  header("HTTP/1.1 302 Found");
  header("Location: ./birthday.php?success=1");
  return;
}
?>
<a href="./index.php">設定一覧に戻る</a>

<h1>生年月日</h1>
<form method="POST">
  <input type="date" name="birthday" value="<?= htmlspecialchars($user['birthday']) ?>">
  <button type="submit">決定</button>
</form>

<?php if(!empty($_GET['success'])): ?>
<div>
  生年月日の変更処理が完了しました。
</div>
<?php endif; ?>
```

4. プロフィールページの作成
vim public/profile.php↓
```
<?php
$user = null;
if (!empty($_GET['user_id'])) {
  $user_id = $_GET['user_id'];
  // DBに接続
  $dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');
  // 対象の会員情報を引く
  $select_sth = $dbh->prepare("SELECT * FROM users WHERE id = :id");
  $select_sth->execute([
      ':id' => $user_id,
  ]);
  $user = $select_sth->fetch();
}
if (empty($user)) {
  header("HTTP/1.1 404 Not Found");
  print("そのようなユーザーIDの会員情報は存在しません");
  return;
}
// 投稿データを取得。紐づく会員情報も結合し同時に取得する。
$select_sth = $dbh->prepare(
  'SELECT bbs_entries.*, users.name AS user_name, users.icon_filename AS user_icon_filename'
  . ' FROM bbs_entries INNER JOIN users ON bbs_entries.user_id = users.id'
  . ' WHERE user_id = :user_id'
  . ' ORDER BY bbs_entries.created_at DESC'
);
$select_sth->execute([
  ':user_id' => $user_id,
]);
?>
<a href="/bbs.php">掲示板に戻る</a>
<div style="
    width: 100%; height: 15em;
    <?php if(!empty($user['cover_filename'])): ?>
    background: url('/image/<?= $user['cover_filename'] ?>') center;
    background-size: cover;
    <?php endif; ?>
  ">
</div>
<h1><?= htmlspecialchars($user['name']) ?> さん のプロフィール</h1>
<div>
  <?php if(empty($user['icon_filename'])): ?>
  現在未設定
  <?php else: ?>
  <img src="/image/<?= $user['icon_filename'] ?>"
    style="height: 5em; width: 5em; border-radius: 50%; object-fit: cover;">
  <?php endif; ?>
</div>

<?php if(!empty($user['birthday'])): ?>
<?php
  $birthday = DateTime::createFromFormat('Y-m-d', $user['birthday']);
  $today = new DateTime('now');
?>
  <?= $today->diff($birthday)->y ?>歳
<?php endif; ?>

<div>
  <?= nl2br(htmlspecialchars($user['introduction'] ?? '')) ?>
</div>
<hr>
<?php foreach($select_sth as $entry): ?>
  <dl style="margin-bottom: 1em; padding-bottom: 1em; border-bottom: 1px solid #ccc;">
    <dt>日時</dt>
    <dd><?= $entry['created_at'] ?></dd>
    <dt>内容</dt>
    <dd>
      <?= htmlspecialchars($entry['body']) ?>
      <?php if(!empty($entry['image_filename'])): ?>
      <div>
        <img src="/image/<?= $entry['image_filename'] ?>" style="max-height: 10em;">
      </div>
      <?php endif; ?>
    </dd>
  </dl>
<?php endforeach ?>
```



**会員サービスに紐づけた掲示板の作成**
1. 新しく掲示板投稿テーブルを作成
```
DROP TABLE bbs_entries;
```

```
CREATE TABLE `bbs_entries` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    `user_id` INT UNSIGNED NOT NULL,
    `body` TEXT NOT NULL,
    `image_filename` TEXT DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

vim public/bbs.php↓
```
<?php
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');

session_start();
if (isset($_POST['body']) && !empty($_SESSION['login_user_id'])) {
  // POSTで送られてくるフォームパラメータ body がある かつ ログイン状態 の場合

  $image_filename = null;
  if (!empty($_POST['image_base64'])) {
    // 先頭の data:~base64, のところは削る
    $base64 = preg_replace('/^data:.+base64,/', '', $_POST['image_base64']);

    // base64からバイナリにデコードする
    $image_binary = base64_decode($base64);

    // 新しいファイル名を決めてバイナリを出力する
    $image_filename = strval(time()) . bin2hex(random_bytes(25)) . '.png';
    $filepath =  '/var/www/upload/image/' . $image_filename;
    file_put_contents($filepath, $image_binary);
  }

  // insertする
  $insert_sth = $dbh->prepare("INSERT INTO bbs_entries (user_id, body, image_filename) VALUES (:user_id, :body, :image_filename)");
  $insert_sth->execute([
      ':user_id' => $_SESSION['login_user_id'], // ログインしている会員情報の主キー
      ':body' => $_POST['body'], // フォームから送られてきた投稿本文
      ':image_filename' => $image_filename, // 保存した画像の名前 (nullの場合もある)
  ]);

  // 処理が終わったらリダイレクトする
  // リダイレクトしないと，リロード時にまた同じ内容でPOSTすることになる
  header("HTTP/1.1 302 Found");
  header("Location: ./bbs.php");
  return;
}

// いままで保存してきたものを取得
$select_sth = $dbh->prepare('SELECT * FROM bbs_entries ORDER BY created_at DESC');
$select_sth->execute();

// bodyのHTMLを出力するための関数を用意する
function bodyFilter (string $body): string
{
    $body = htmlspecialchars($body); // エスケープ処理
    $body = nl2br($body); // 改行文字を<br>要素に変換

    // >>1 といった文字列を該当番号の投稿へのページ内リンクとする (レスアンカー機能)
    // 「>」(半角の大なり記号)は htmlspecialchars() でエスケープされているため注意
    $body = preg_replace('/&gt;&gt;(\d+)/', '<a href="#entry$1">&gt;&gt;$1</a>', $body);

    return $body;
}
?>

<?php if(empty($_SESSION['login_user_id'])): ?>
  投稿するには<a href="/login.php">ログイン</a>が必要です。
<?php else: ?>
  <!-- フォームのPOST先はこのファイル自身にする -->
  <form method="POST" action="./bbs.php"><!-- enctypeは外しておきましょう -->
    <textarea name="body" required></textarea>
    <div style="margin: 1em 0;">
      <input type="file" accept="image/*" name="image" id="imageInput">
    </div>
    <input id="imageBase64Input" type="hidden" name="image_base64"><!-- base64を送る用のinput (非表示) -->
    <canvas id="imageCanvas" style="display: none;"></canvas><!-- 画像縮小に使うcanvas (非表示) -->
    <button type="submit">送信</button>
  </form>
<?php endif; ?>
<hr>

<?php foreach($select_sth as $entry): ?>
  <dl style="margin-bottom: 1em; padding-bottom: 1em; border-bottom: 1px solid #ccc;">
    <dt id="entry<?= htmlspecialchars($entry['id']) ?>">
      番号
    </dt>
    <dd>
      <?= htmlspecialchars($entry['id']) ?>
    </dd>
    <dt>
      投稿者
    </dt>
    <dd>
      会員ID: <?= htmlspecialchars($entry['user_id']) ?>
    </dd>
    <dt>日時</dt>
    <dd><?= $entry['created_at'] ?></dd>
    <dt>内容</dt>
    <dd>
      <?= bodyFilter($entry['body']) ?>
      <?php if(!empty($entry['image_filename'])): ?>
      <div>
        <img src="/image/<?= $entry['image_filename'] ?>" style="max-height: 10em;">
      </div>
      <?php endif; ?>
    </dd>
  </dl>
<?php endforeach ?>

<script>
document.addEventListener("DOMContentLoaded", () => {
  const imageInput = document.getElementById("imageInput");
  imageInput.addEventListener("change", () => {
    if (imageInput.files.length < 1) {
      // 未選択の場合
      return;
    }

    const file = imageInput.files[0];
    if (!file.type.startsWith('image/')){ // 画像でなければスキップ
      return;
    }

    // 画像縮小処理
    const imageBase64Input = document.getElementById("imageBase64Input"); // base64を送るようのinput
    const canvas = document.getElementById("imageCanvas"); // 描画するcanvas
    const reader = new FileReader();
    const image = new Image();
    reader.onload = () => { // ファイルの読み込み完了したら動く処理を指定
      image.onload = () => { // 画像として読み込み完了したら動く処理を指定

        // 元の縦横比を保ったまま縮小するサイズを決めてcanvasの縦横に指定する
        const originalWidth = image.naturalWidth; // 元画像の横幅
        const originalHeight = image.naturalHeight; // 元画像の高さ
        const maxLength = 1000; // 横幅も高さも1000以下に縮小するものとする
        if (originalWidth <= maxLength && originalHeight <= maxLength) { // どちらもmaxLength以下の場合そのまま
            canvas.width = originalWidth;
            canvas.height = originalHeight;
        } else if (originalWidth > originalHeight) { // 横長画像の場合
            canvas.width = maxLength;
            canvas.height = maxLength * originalHeight / originalWidth;
        } else { // 縦長画像の場合
            canvas.width = maxLength * originalWidth / originalHeight;
            canvas.height = maxLength;
        }

        // canvasに実際に画像を描画 (canvasはdisplay:noneで隠れているためわかりにくいが...)
        const context = canvas.getContext("2d");
        context.drawImage(image, 0, 0, canvas.width, canvas.height);

        // canvasの内容をbase64に変換しinputのvalueに設定
        imageBase64Input.value = canvas.toDataURL();
      };
      image.src = reader.result;
    };
    reader.readAsDataURL(file);
  });
});
</script>
```

2. ログインや会員登録フォームまわりの導線を整える
vim public/login.php
```diff
<h1>ログイン</h1>

+ 初めての人は<a href="/signup.php">会員登録</a>しましょう。
+ <hr>
+ 
<!-- ログインフォーム -->
<form method="POST">
  <!-- input要素のtype属性は全部textでも動くが、適切なものに設定すると利用者は使いやすい -->
```
vim public/login_finish.php
```diff
<h1>ログイン完了</h1>

<p>
-   ログイン完了しました!
+   ログイン完了しました!<br>
+   <a href="/bbs.php">掲示板はこちら</a>
</p>
<hr>
<p>
```
vim public/signup.php
```diff

<h1>会員登録</h1>

+ 会員登録済の人は<a href="/login.php">ログイン</a>しましょう。
+ <hr>
+ 
<!-- 登録フォーム -->
<form method="POST">
  <!-- input要素のtype属性は全部textでも動くが、適切なものに設定すると利用者は使いやすい -->
```
vim public/signup_finish.php
```diff
<h1>会員登録完了</h1>

- 会員登録が完了しました。
+ 会員登録が完了しました。<br>
+ 登録した内容をもとに<a href="/login.php">ログイン</a>してください。
```

3. 掲示板の投稿で会員情報も同時に表示する
vim public/bbs.php
```diff
  return;
}

- // いままで保存してきたものを取得
- $select_sth = $dbh->prepare('SELECT * FROM bbs_entries ORDER BY created_at DESC');
+ // 投稿データを取得。紐づく会員情報も結合し同時に取得する。
+ $select_sth = $dbh->prepare(
+   'SELECT bbs_entries.*, users.name AS user_name'
+   . ' FROM bbs_entries INNER JOIN users ON bbs_entries.user_id = users.id'
+   . ' ORDER BY bbs_entries.created_at DESC'
+ );
$select_sth->execute();

// bodyのHTMLを出力するための関数を用意する
```
```diff
      投稿者
    </dt>
    <dd>
-       会員ID: <?= htmlspecialchars($entry['user_id']) ?>
+       <?= htmlspecialchars($entry['user_name']) ?>
+       (ID: <?= htmlspecialchars($entry['user_id']) ?>)
    </dd>
    <dt>日時</dt>
    <dd><?= $entry['created_at'] ?></dd>
```
4. 掲示板の各投稿の横にアイコンを表示
vim public/bbs.php
```diff
// 投稿データを取得。紐づく会員情報も結合し同時に取得する。
$select_sth = $dbh->prepare(
-   'SELECT bbs_entries.*, users.name AS user_name'
+   'SELECT bbs_entries.*, users.name AS user_name, users.icon_filename AS user_icon_filename'
  . ' FROM bbs_entries INNER JOIN users ON bbs_entries.user_id = users.id'
  . ' ORDER BY bbs_entries.created_at DESC'
);
```
```diff
      投稿者
    </dt>
    <dd>
+       <?php if(!empty($entry['user_icon_filename'])): // アイコン画像がある場合は表示 ?>
+       <img src="/image/<?= $entry['user_icon_filename'] ?>"
+         style="height: 2em; width: 2em; border-radius: 50%; object-fit: cover;">
+       <?php endif; ?>
+ 
      <?= htmlspecialchars($entry['user_name']) ?>
      (ID: <?= htmlspecialchars($entry['user_id']) ?>)
    </dd>
```
5. 投稿の一覧から投稿者のプロフィールページに行けるように & プロフィールから掲示板に戻れるように
vim public/bbs.php
```diff
      投稿者
    </dt>
    <dd>
-       <?php if(!empty($entry['user_icon_filename'])): // アイコン画像がある場合は表示 ?>
-       <img src="/image/<?= $entry['user_icon_filename'] ?>"
-         style="height: 2em; width: 2em; border-radius: 50%; object-fit: cover;">
-       <?php endif; ?>
- 
-       <?= htmlspecialchars($entry['user_name']) ?>
-       (ID: <?= htmlspecialchars($entry['user_id']) ?>)
+       <a href="/profile.php?user_id=<?= $entry['user_id'] ?>">
+         <?php if(!empty($entry['user_icon_filename'])): // アイコン画像がある場合は表示 ?>
+         <img src="/image/<?= $entry['user_icon_filename'] ?>"
+           style="height: 2em; width: 2em; border-radius: 50%; object-fit: cover;">
+         <?php endif; ?>
+ 
+         <?= htmlspecialchars($entry['user_name']) ?>
+         (ID: <?= htmlspecialchars($entry['user_id']) ?>)
+       </a>
    </dd>
    <dt>日時</dt>
    <dd><?= $entry['created_at'] ?></dd>
```
vim public/profile.php
```diff
  return;
}
?>
+ <a href="/bbs.php">掲示板に戻る</a>

<h1><?= htmlspecialchars($user['name']) ?> さん のプロフィール</h1>
```
6. プロフィールページにその人の投稿を表示
```diff
  print("そのようなユーザーIDの会員情報は存在しません");
  return;
}
+ 
+ // 投稿データを取得。紐づく会員情報も結合し同時に取得する。
+ $select_sth = $dbh->prepare(
+   'SELECT bbs_entries.*, users.name AS user_name, users.icon_filename AS user_icon_filename'
+   . ' FROM bbs_entries INNER JOIN users ON bbs_entries.user_id = users.id'
+   . ' WHERE user_id = :user_id'
+   . ' ORDER BY bbs_entries.created_at DESC'
+ );
+ $select_sth->execute([
+   ':user_id' => $user_id,
+ ]);
?>
<a href="/bbs.php">掲示板に戻る</a>
```
```diff
<div>
  <?= nl2br(htmlspecialchars($user['introduction'] ?? '')) ?>
</div>
+ 
+ <hr>
+ 
+ <?php foreach($select_sth as $entry): ?>
+   <dl style="margin-bottom: 1em; padding-bottom: 1em; border-bottom: 1px solid #ccc;">
+     <dt>日時</dt>
+     <dd><?= $entry['created_at'] ?></dd>
+     <dt>内容</dt>
+     <dd>
+       <?= htmlspecialchars($entry['body']) ?>
+       <?php if(!empty($entry['image_filename'])): ?>
+       <div>
+         <img src="/image/<?= $entry['image_filename'] ?>" style="max-height: 10em;">
+       </div>
+       <?php endif; ?>
+     </dd>
+   </dl>
+ <?php endforeach ?>
```

6. タイムアウトの時間を延ばす
vim php.ini
```diff
session.save_handler = redis
session.save_path = "tcp://redis:6379"
+ session.gc_maxlifetime = 86400
```


フォロー機能の作成
```
CREATE TABLE `user_relationships` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    `followee_user_id` INT UNSIGNED NOT NULL,
    `follower_user_id` INT UNSIGNED NOT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
);
```
1. フォロー画面の作成
vim public/follow.php
```
<?php
session_start();

if (empty($_SESSION['login_user_id'])) {
  header("HTTP/1.1 302 Found");
  header("Location: ./login.php");
  return;
}

// DBに接続
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');

// フォロー対象(フォローされる側)のデータを引く
$followee_user = null;
if (!empty($_GET['followee_user_id'])) {
  $select_sth = $dbh->prepare("SELECT * FROM users WHERE id = :id");
  $select_sth->execute([
      ':id' => $_GET['followee_user_id'],
  ]);
  $followee_user = $select_sth->fetch();
}
if (empty($followee_user)) {
  header("HTTP/1.1 404 Not Found");
  print("そのようなユーザーIDの会員情報は存在しません");
  return;
}

// 現在のフォロー状態をDBから取得
$select_sth = $dbh->prepare(
  "SELECT * FROM user_relationships"
  . " WHERE follower_user_id = :follower_user_id AND followee_user_id = :followee_user_id"
);
$select_sth->execute([
    ':followee_user_id' => $followee_user['id'], // フォローされる側(フォロー対象)
    ':follower_user_id' => $_SESSION['login_user_id'], // フォローする側はログインしている会員
]);
$relationship = $select_sth->fetch();
if (!empty($relationship)) { // 既にフォロー関係がある場合は適当なエラー表示して終了
  print("既にフォローしています。");
  return;
}

$insert_result = false;
if ($_SERVER['REQUEST_METHOD'] == 'POST') { // フォームでPOSTした場合は実際のフォロー登録処理を行う
  $insert_sth = $dbh->prepare(
    "INSERT INTO user_relationships (follower_user_id, followee_user_id) VALUES (:follower_user_id, :followee_user_id)"
  );
  $insert_result = $insert_sth->execute([
      ':followee_user_id' => $followee_user['id'], // フォローされる側(フォロー対象)
      ':follower_user_id' => $_SESSION['login_user_id'], // フォローする側はログインしている会員
  ]);
}
?>

<?php if($insert_result): ?>
<div>
  <?= htmlspecialchars($followee_user['name']) ?> さんをフォローしました。<br>
  <a href="/profile.php?user_id=<?= $followee_user['id'] ?>">
    <?= htmlspecialchars($followee_user['name']) ?> さんのプロフィールに戻る
  </a>
</div>
<?php else: ?>
<div>
  <?= htmlspecialchars($followee_user['name']) ?> さんをフォローしますか?
  <form method="POST">
    <button type="submit">
      フォローする
    </button>
  </form>
</div>
<?php endif; ?>
```
2. プロフィールページから遷移できるように
vim public/profile.php
```diff
$select_sth->execute([
  ':user_id' => $user_id,
]);
+ 
+ // フォロー状態を取得
+ $relationship = null;
+ session_start();
+ if (!empty($_SESSION['login_user_id'])) { // ログインしている場合
+   // フォロー状態をDBから取得
+   $select_sth = $dbh->prepare(
+     "SELECT * FROM user_relationships"
+     . " WHERE follower_user_id = :follower_user_id AND followee_user_id = :followee_user_id"
+   );
+   $select_sth->execute([
+       ':followee_user_id' => $user['id'], // フォローされる側は閲覧しようとしているプロフィールの会員
+       ':follower_user_id' => $_SESSION['login_user_id'], // フォローする側はログインしている会員
+   ]);
+   $relationship = $select_sth->fetch();
}
?>
<a href="/bbs.php">掲示板に戻る</a>
```

```diff
  <?php endif; ?>
</div>

+ <?php if(empty($relationship)): // フォローしていない場合 ?>
+ <div>
+   <a href="./follow.php?followee_user_id=<?= $user['id'] ?>">フォローする</a>
+ </div>
+ <?php else: // フォローしている場合 ?>
+ <div>
+   <?= $relationship['created_at'] ?> にフォローしました。
+ </div>
+ <?php endif; ?>

<?php if(!empty($user['birthday'])): ?>
<?php
  $birthday = DateTime::createFromFormat('Y-m-d', $user['birthday']);
```

3.フォロー一覧
vim public/follow_list.php 
```
<?php
session_start();

// ログインしてなければログイン画面に飛ばす
if (empty($_SESSION['login_user_id'])) {
  header("HTTP/1.1 302 Found");
  header("Location: /login.php");
  return;
}

// DBに接続
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');

// 自分がフォローしている一覧をDBから引く。
// テーブル結合を使って、フォローしている対象の会員情報も一緒に取得。
$select_sth = $dbh->prepare(
  'SELECT user_relationships.*, users.name AS followee_user_name, users.icon_filename AS followee_user_icon_filename'
  . ' FROM user_relationships INNER JOIN users ON user_relationships.followee_user_id = users.id'
  . ' WHERE user_relationships.follower_user_id = :follower_user_id'
  . ' ORDER BY user_relationships.id DESC'
);
$select_sth->execute([
    ':follower_user_id' => $_SESSION['login_user_id'],
]);
?>

<h1>フォローしている一覧</h1>

<ul>
  <?php foreach($select_sth as $relationship): ?>
  <li>
    <a href="/profile.php?user_id=<?= $relationship['followee_user_id'] ?>">
      <?php if(!empty($relationship['followee_user_icon_filename'])): // アイコン画像がある場合は表示 ?>
      <img src="/image/<?= $relationship['followee_user_icon_filename'] ?>"
        style="height: 2em; width: 2em; border-radius: 50%; object-fit: cover;">
      <?php endif; ?>

      <?= htmlspecialchars($relationship['followee_user_name']) ?>
      (ID: <?= htmlspecialchars($relationship['followee_user_id']) ?>)
    </a>
    (<?= $relationship['created_at'] ?>にフォロー)
  </li>
  <?php endforeach; ?>
</ul>
```

4. フォロワー一覧
vim public/follower_list.php
```
<?php
session_start();

// ログインしてなければログイン画面に飛ばす
if (empty($_SESSION['login_user_id'])) {
  header("HTTP/1.1 302 Found");
  header("Location: /login.php");
  return;
}

// DBに接続
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');

// 自分がフォローされている一覧をDBから引く。
// テーブル結合を使って、フォローしている対象の会員情報も一緒に取得。
$select_sth = $dbh->prepare(
  'SELECT user_relationships.*, users.name AS follower_user_name, users.icon_filename AS follower_user_icon_filename'
  . ' FROM user_relationships INNER JOIN users ON user_relationships.follower_user_id = users.id'
  . ' WHERE user_relationships.followee_user_id = :followee_user_id'
  . ' ORDER BY user_relationships.id DESC'
);
$select_sth->execute([
    ':followee_user_id' => $_SESSION['login_user_id'],
]);
?>

<h1>フォローされている一覧</h1>

<ul>
  <?php foreach($select_sth as $relationship): ?>
  <li>
    <a href="/profile.php?user_id=<?= $relationship['follower_user_id'] ?>">
      <?php if(!empty($relationship['follower_user_icon_filename'])): // アイコン画像がある場合は表示 ?>
      <img src="/image/<?= $relationship['follower_user_icon_filename'] ?>"
        style="height: 2em; width: 2em; border-radius: 50%; object-fit: cover;">
      <?php endif; ?>

      <?= htmlspecialchars($relationship['follower_user_name']) ?>
      (ID: <?= htmlspecialchars($relationship['follower_user_id']) ?>)
    </a>
    (<?= $relationship['created_at'] ?>にフォローされました)
  </li>
  <?php endforeach; ?>
</ul>
```

5. フォロワーのプロフィールページに、フォローされている旨を表示
vim public/profile.php
```diff
  ]);
  $relationship = $select_sth->fetch();
}
+ 
+ // フォローされている状態を取得
+ $follower_relationship = null;
+ if (!empty($_SESSION['login_user_id'])) { // ログインしている場合
+   // フォローされている状態をDBから取得
+   $select_sth = $dbh->prepare(
+     "SELECT * FROM user_relationships"
+     . " WHERE follower_user_id = :follower_user_id AND followee_user_id = :followee_user_id"
+   );
+   $select_sth->execute([
+       ':follower_user_id' => $user['id'], // フォローしている側は閲覧しようとしているプロフィールの会員
+       ':followee_user_id' => $_SESSION['login_user_id'], // フォローされる側はログインしている会員
+   ]);
+   $follower_relationship = $select_sth->fetch();
+ }
?>
<a href="/bbs.php">掲示板に戻る</a>
```

```diff
</div>
<?php endif; ?>

+ <?php if(!empty($follower_relationship)): // フォローされている場合 ?>
+ <div>
+   フォローされています。
+ </div>
+ <?php endif; ?>
+ 
<?php if(!empty($user['birthday'])): ?>
<?php
  $birthday = DateTime::createFromFormat('Y-m-d', $user['birthday']);
```


タイムラインの作成
vim public/timeline.php
```
<?php
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');

session_start();
if (empty($_SESSION['login_user_id'])) { // 非ログインの場合利用不可
  header("HTTP/1.1 302 Found");
  header("Location: /login.php");
  return;
}

// 投稿処理
if (isset($_POST['body']) && !empty($_SESSION['login_user_id'])) {

  $image_filename = null;
  if (!empty($_POST['image_base64'])) {
    // 先頭の data:~base64, のところは削る
    $base64 = preg_replace('/^data:.+base64,/', '', $_POST['image_base64']);

    // base64からバイナリにデコードする
    $image_binary = base64_decode($base64);

    // 新しいファイル名を決めてバイナリを出力する
    $image_filename = strval(time()) . bin2hex(random_bytes(25)) . '.png';
    $filepath =  '/var/www/upload/image/' . $image_filename;
    file_put_contents($filepath, $image_binary);
  }

  // insertする
  $insert_sth = $dbh->prepare("INSERT INTO bbs_entries (user_id, body, image_filename) VALUES (:user_id, :body, :image_filename)");
  $insert_sth->execute([
      ':user_id' => $_SESSION['login_user_id'], // ログインしている会員情報の主キー
      ':body' => $_POST['body'], // フォームから送られてきた投稿本文
      ':image_filename' => $image_filename, // 保存した画像の名前 (nullの場合もある)
  ]);

  // 処理が終わったらリダイレクトする
  // リダイレクトしないと，リロード時にまた同じ内容でPOSTすることになる
  header("HTTP/1.1 302 Found");
  header("Location: ./bbs.php");
  return;
}

// 投稿データを取得。
// フォローしている人の投稿と自分自身の投稿のみ表示
$sql = 'SELECT bbs_entries.*, users.name AS user_name, users.icon_filename AS user_icon_filename'
  . ' FROM bbs_entries'
  . ' INNER JOIN users ON bbs_entries.user_id = users.id'
  . ' LEFT OUTER JOIN user_relationships ON bbs_entries.user_id = user_relationships.followee_user_id'
  . ' WHERE user_relationships.follower_user_id = :login_user_id OR bbs_entries.user_id = :login_user_id'
  . ' ORDER BY bbs_entries.created_at DESC';
$select_sth = $dbh->prepare($sql);
$select_sth->execute([
    ':login_user_id' => $_SESSION['login_user_id'],
]);

// bodyのHTMLを出力するための関数を用意する
function bodyFilter (string $body): string
{
    $body = htmlspecialchars($body); // エスケープ処理
    $body = nl2br($body); // 改行文字を<br>要素に変換

    // >>1 といった文字列を該当番号の投稿へのページ内リンクとする (レスアンカー機能)
    // 「>」(半角の大なり記号)は htmlspecialchars() でエスケープされているため注意
    $body = preg_replace('/&gt;&gt;(\d+)/', '<a href="#entry$1">&gt;&gt;$1</a>', $body);

    return $body;
}
?>

<?php if(empty($_SESSION['login_user_id'])): ?>
  投稿するには<a href="/login.php">ログイン</a>が必要です。
<?php else: ?>
  現在ログイン中 (<a href="/setting/index.php">設定画面はこちら</a>)
  <!-- フォームのPOST先はこのファイル自身にする -->
  <form method="POST" action="./bbs.php"><!-- enctypeは外しておきましょう -->
    <textarea name="body" required></textarea>
    <div style="margin: 1em 0;">
      <input type="file" accept="image/*" name="image" id="imageInput">
    </div>
    <input id="imageBase64Input" type="hidden" name="image_base64"><!-- base64を送る用のinput (非表示) -->
    <canvas id="imageCanvas" style="display: none;"></canvas><!-- 画像縮小に使うcanvas (非表示) -->
    <button type="submit">送信</button>
  </form>
<?php endif; ?>
<hr>

<?php foreach($select_sth as $entry): ?>
  <dl style="margin-bottom: 1em; padding-bottom: 1em; border-bottom: 1px solid #ccc;">
    <dt id="entry<?= htmlspecialchars($entry['id']) ?>">
      番号
    </dt>
    <dd>
      <?= htmlspecialchars($entry['id']) ?>
    </dd>
    <dt>
      投稿者
    </dt>
    <dd>
      <a href="/profile.php?user_id=<?= $entry['user_id'] ?>">
        <?php if(!empty($entry['user_icon_filename'])): // アイコン画像がある場合は表示 ?>
        <img src="/image/<?= $entry['user_icon_filename'] ?>"
          style="height: 2em; width: 2em; border-radius: 50%; object-fit: cover;">
        <?php endif; ?>

        <?= htmlspecialchars($entry['user_name']) ?>
        (ID: <?= htmlspecialchars($entry['user_id']) ?>)
      </a>
    </dd>
    <dt>日時</dt>
    <dd><?= $entry['created_at'] ?></dd>
    <dt>内容</dt>
    <dd>
      <?= bodyFilter($entry['body']) ?>
      <?php if(!empty($entry['image_filename'])): ?>
      <div>
        <img src="/image/<?= $entry['image_filename'] ?>" style="max-height: 10em;">
      </div>
      <?php endif; ?>
    </dd>
  </dl>
<?php endforeach ?>

<script>
document.addEventListener("DOMContentLoaded", () => {
  const imageInput = document.getElementById("imageInput");
  imageInput.addEventListener("change", () => {
    if (imageInput.files.length < 1) {
      // 未選択の場合
      return;
    }

    const file = imageInput.files[0];
    if (!file.type.startsWith('image/')){ // 画像でなければスキップ
      return;
    }

    // 画像縮小処理
    const imageBase64Input = document.getElementById("imageBase64Input"); // base64を送るようのinput
    const canvas = document.getElementById("imageCanvas"); // 描画するcanvas
    const reader = new FileReader();
    const image = new Image();
    reader.onload = () => { // ファイルの読み込み完了したら動く処理を指定
      image.onload = () => { // 画像として読み込み完了したら動く処理を指定

        // 元の縦横比を保ったまま縮小するサイズを決めてcanvasの縦横に指定する
        const originalWidth = image.naturalWidth; // 元画像の横幅
        const originalHeight = image.naturalHeight; // 元画像の高さ
        const maxLength = 1000; // 横幅も高さも1000以下に縮小するものとする
        if (originalWidth <= maxLength && originalHeight <= maxLength) { // どちらもmaxLength以下の場合そのまま
            canvas.width = originalWidth;
            canvas.height = originalHeight;
        } else if (originalWidth > originalHeight) { // 横長画像の場合
            canvas.width = maxLength;
            canvas.height = maxLength * originalHeight / originalWidth;
        } else { // 縦長画像の場合
            canvas.width = maxLength * originalWidth / originalHeight;
            canvas.height = maxLength;
        }

        // canvasに実際に画像を描画 (canvasはdisplay:noneで隠れているためわかりにくいが...)
        const context = canvas.getContext("2d");
        context.drawImage(image, 0, 0, canvas.width, canvas.height);

        // canvasの内容をbase64に変換しinputのvalueに設定
        imageBase64Input.value = canvas.toDataURL();
      };
      image.src = reader.result;
    };
    reader.readAsDataURL(file);
  });
});
</script>
```


会員一覧画面を作る
vim public/users.php
```
<?php

$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');

// 会員データを取得
$select_sth = $dbh->prepare('SELECT * FROM users ORDER BY id DESC');
$select_sth->execute();
?>

<body>
  <h1>会員一覧</h1>

  <?php foreach($select_sth as $user): ?>
    <div style="display: flex; justify-content: start; align-items: center; padding: 1em 2em;">
      <?php if(empty($user['icon_filename'])): ?>
        <!-- アイコン無い場合は同じ大きさの空白を表示して揃えておく -->
        <div style="height: 2em; width: 2em;"></div>
      <?php else: ?>
        <img src="/image/<?= $user['icon_filename'] ?>"
          style="height: 2em; width: 2em; border-radius: 50%; object-fit: cover;">
      <?php endif; ?>
      <a href="/profile.php?user_id=<?= $user['id'] ?>" style="margin-left: 1em;">
        <?= htmlspecialchars($user['name']) ?>
      </a>
    </div>
    <hr style="border: none; border-bottom: 1px solid gray;">
  <?php endforeach; ?>
</body>
```

同線管理
1. 掲示板へのリンクをタイムラインへのリンクに変更
vim public/login_finish.php
```diff
<p>
  ログイン完了しました!<br>
-   <a href="/bbs.php">掲示板はこちら</a>
+   <a href="/timeline.php">タイムラインはこちら</a>
</p>
<hr>
<p>
```
vim puclic/profile.php
```diff
  $follower_relationship = $select_sth->fetch();
}
?>
- <a href="/bbs.php">掲示板に戻る</a>
+ <a href="/timeline.php">タイムラインに戻る</a>

<div style="
    width: 100%; height: 15em;
```
vim public/setting/index.php
```diff
$user = $select_sth->fetch();
?>

- <a href="/bbs.php">掲示板に戻る</a>
+ <a href="/timeline.php">タイムラインに戻る</a>

<h1>設定画面</h1>

```
vim public/timeline.php
```diff
  // 処理が終わったらリダイレクトする
  // リダイレクトしないと，リロード時にまた同じ内容でPOSTすることになる
  header("HTTP/1.1 302 Found");
-   header("Location: ./bbs.php");
+   header("Location: ./timeline.php");
  return;
}
```
```diff
<?php else: ?>
  現在ログイン中 (<a href="/setting/index.php">設定画面はこちら</a>)
  <!-- フォームのPOST先はこのファイル自身にする -->
-   <form method="POST" action="./bbs.php"><!-- enctypeは外しておきましょう -->
+   <form method="POST" action="./timeline.php"><!-- enctypeは外しておきましょう -->
    <textarea name="body" required></textarea>
    <div style="margin: 1em 0;">
      <input type="file" accept="image/*" name="image" id="imageInput">
```

2. 掲示板の投稿機能は削除
vim public/bbs.php
```
<?php
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');

session_start();

// 投稿データを取得。紐づく会員情報も結合し同時に取得する。
$select_sth = $dbh->prepare(
  'SELECT bbs_entries.*, users.name AS user_name, users.icon_filename AS user_icon_filename'
  . ' FROM bbs_entries INNER JOIN users ON bbs_entries.user_id = users.id'
  . ' ORDER BY bbs_entries.created_at DESC'
);
$select_sth->execute();
// bodyのHTMLを出力するための関数を用意する
function bodyFilter (string $body): string
{
    $body = htmlspecialchars($body); // エスケープ処理
    $body = nl2br($body); // 改行文字を<br>要素に変換
    // >>1 といった文字列を該当番号の投稿へのページ内リンクとする (レスアンカー機能)
    // 「>」(半角の大なり記号)は htmlspecialchars() でエスケープされているため注意
    $body = preg_replace('/&gt;&gt;(\d+)/', '<a href="#entry$1">&gt;&gt;$1</a>', $body);
    return $body;
}
?>

<?php if(empty($_SESSION['login_user_id'])): ?>
  <a href="/login.php">ログイン</a>して自分のタイムラインを閲覧しましょう！
<?php else: ?>
  <a href="/timeline.php">タイムラインはこちら</a>
<?php endif; ?>
<hr>

<?php foreach($select_sth as $entry): ?>
  <dl style="margin-bottom: 1em; padding-bottom: 1em; border-bottom: 1px solid #ccc;">
    <dt id="entry<?= htmlspecialchars($entry['id']) ?>">
      番号
    </dt>
    <dd>
      <?= htmlspecialchars($entry['id']) ?>
    </dd>
    <dt>
      投稿者
    </dt>
    <dd>
      <a href="/profile.php?user_id=<?= $entry['user_id'] ?>">
        <?php if(!empty($entry['user_icon_filename'])): // アイコン画像がある場合は表示 ?>
        <img src="/image/<?= $entry['user_icon_filename'] ?>"
          style="height: 2em; width: 2em; border-radius: 50%; object-fit: cover;">
        <?php endif; ?>
        <?= htmlspecialchars($entry['user_name']) ?>
        (ID: <?= htmlspecialchars($entry['user_id']) ?>)
      </a>
    </dd>
    <dt>日時</dt>
    <dd><?= $entry['created_at'] ?></dd>
    <dt>内容</dt>
    <dd>
      <?= bodyFilter($entry['body']) ?>
      <?php if(!empty($entry['image_filename'])): ?>
      <div>
        <img src="/image/<?= $entry['image_filename'] ?>" style="max-height: 10em;">
      </div>
      <?php endif; ?>
    </dd>
  </dl>
<?php endforeach ?>
```

3. タイムラインにログイン状態表示
vim public/timeline.php
```diff
  return;
}

+ // 現在のログイン情報を取得する
+ $user_select_sth = $dbh->prepare("SELECT * from users WHERE id = :id");
+ $user_select_sth->execute([':id' => $_SESSION['login_user_id']]);
+ $user = $user_select_sth->fetch();
+ 
// 投稿処理
if (isset($_POST['body']) && !empty($_SESSION['login_user_id'])) {
```

```diff
}
?>

- <?php if(empty($_SESSION['login_user_id'])): ?>
-   投稿するには<a href="/login.php">ログイン</a>が必要です。
- <?php else: ?>
-   現在ログイン中 (<a href="/setting/index.php">設定画面はこちら</a>)
-   <!-- フォームのPOST先はこのファイル自身にする -->
-   <form method="POST" action="./timeline.php"><!-- enctypeは外しておきましょう -->
-     <textarea name="body" required></textarea>
-     <div style="margin: 1em 0;">
-       <input type="file" accept="image/*" name="image" id="imageInput">
-     </div>
-     <input id="imageBase64Input" type="hidden" name="image_base64"><!-- base64を送る用のinput (非表示) -->
-     <canvas id="imageCanvas" style="display: none;"></canvas><!-- 画像縮小に使うcanvas (非表示) -->
-     <button type="submit">送信</button>
-   </form>
- <?php endif; ?>
- <hr>
+ <div>
+   現在 <?= htmlspecialchars($user['name']) ?> (ID: <?= $user['id'] ?>) さんでログイン中
+ </div>
+ <div style="margin-bottom: 1em;">
+   <a href="/setting/index.php">設定画面</a>
+   /
+   <a href="/users.php">会員一覧画面</a>
+ </div>

+ <!-- フォームのPOST先はこのファイル自身にする -->
+ <form method="POST" action="./timeline.php"><!-- enctypeは外しておきましょう -->
+   <textarea name="body" required></textarea>
+   <div style="margin: 1em 0;">
+     <input type="file" accept="image/*" name="image" id="imageInput">
+   </div>
+   <input id="imageBase64Input" type="hidden" name="image_base64"><!-- base64を送る用のinput (非表示) -->
+   <canvas id="imageCanvas" style="display: none;"></canvas><!-- 画像縮小に使うcanvas (非表示) -->
+   <button type="submit">送信</button>
+ </form>

<?php foreach($select_sth as $entry): ?>
  <dl style="margin-bottom: 1em; padding-bottom: 1em; border-bottom: 1px solid #ccc;">
```
4. 会員一覧画面にも同様に、設定画面/タイムラインへ遷移しやすいようにしましょう
vim public/users.php
```diff
<body>
  <h1>会員一覧</h1>

+   <div style="margin-bottom: 1em;">
+     <a href="/setting/index.php">設定画面</a>
+     /
+     <a href="/timeline.php">タイムライン</a>
+   </div>

  <?php foreach($select_sth as $user): ?>
    <div style="display: flex; justify-content: start; align-items: center; padding: 1em 2em;">
      <?php if(empty($user['icon_filename'])): ?>
```
5. 自分自身のプロフィールを開いているときに、設定画面に遷移できるようにしてみましょう。また、自分自身に対するフォローボタンなどは非表示にするとよいですね。
vim public/profile.php
```diff
  <?php endif; ?>
</div>

- <?php if(empty($relationship)): // フォローしていない場合 ?>
- <div>
-   <a href="./follow.php?followee_user_id=<?= $user['id'] ?>">フォローする</a>
- </div>
- <?php else: // フォローしている場合 ?>
- <div>
-   <?= $relationship['created_at'] ?> にフォローしました。
+ <?php if($user['id'] === $_SESSION['login_user_id']): // 自分自身の場合 ?>
+ <div style="margin: 1em 0;">
+   これはあなたです！<br>
+   <a href="/setting/index.php">設定画面はこちら</a>
</div>
- <?php endif; ?>
- 
- <?php if(!empty($follower_relationship)): // フォローされている場合 ?>
- <div>
-   フォローされています。
+ <?php else: // 他人の場合 ?>
+ <div style="margin: 1em 0;">
+   <?php if(empty($relationship)): // フォローしていない場合 ?>
+   <div>
+     <a href="./follow.php?followee_user_id=<?= $user['id'] ?>">フォローする</a>
+   </div>
+   <?php else: // フォローしている場合 ?>
+   <div>
+     <?= $relationship['created_at'] ?> にフォローしました。
+   </div>
+   <?php endif; ?>
+   <?php if(!empty($follower_relationship)): // フォローされている場合 ?>
+   <div>
+     フォローされています。
+   </div>
+   <?php endif; ?>
</div>
<?php endif; ?>
```

6. 直接フォローできるようにする
vim public/follow.php
```diff
<div>
  <?= htmlspecialchars($followee_user['name']) ?> さんをフォローしました。<br>
  <a href="/profile.php?user_id=<?= $followee_user['id'] ?>">
-     <?= htmlspecialchars($followee_user['name']) ?> さんのプロフィールに戻る
+     <?= htmlspecialchars($followee_user['name']) ?> さんのプロフィールへ
+   </a>
+   /
+   <a href="/users.php">
+     会員一覧へ
  </a>
</div>
<?php else: ?>
```

vim public/users.php
```diff
<?php
+ session_start();

$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');

// 会員データを取得
$select_sth = $dbh->prepare('SELECT * FROM users ORDER BY id DESC');
$select_sth->execute();
+ 
+ // ログインしている場合、フォローしている会員IDリストを取得
+ $followee_user_ids = [];
+ if (!empty($_SESSION['login_user_id'])) {
+   $followee_users_select_sth = $dbh->prepare(
+     'SELECT * FROM user_relationships WHERE follower_user_id = :follower_user_id'
+   );
+   $followee_users_select_sth->execute([
+     ':follower_user_id' => $_SESSION['login_user_id'],
+   ]);
+   $followee_user_ids = array_map(
+       function ($relationship) {
+           return $relationship['followee_user_id'];
+       },
+       $followee_users_select_sth->fetchAll()
+   ); // array_map で followee_user_id カラムだけ抜き出す
+ }
?>

<body>
```

```diff
      <a href="/profile.php?user_id=<?= $user['id'] ?>" style="margin-left: 1em;">
        <?= htmlspecialchars($user['name']) ?>
      </a>
+       <div style="margin-left: 2em;">
+         <?php if($user['id'] === $_SESSION['login_user_id']): ?>
+           これはあなたです!
+         <?php elseif(in_array($user['id'], $followee_user_ids)): ?>
+           フォロー済
+         <?php else: ?>
+           <a href="./follow.php?followee_user_id=<?= $user['id'] ?>">フォローする</a>
+         <?php endif; ?>
+       </div>
    </div>
    <hr style="border: none; border-bottom: 1px solid gray;">
  <?php endforeach; ?>
```

7. 名前で検索
vim public/users.php
```diff
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');

// 会員データを取得
- $select_sth = $dbh->prepare('SELECT * FROM users ORDER BY id DESC');
- $select_sth->execute();
+ $sql = 'SELECT * FROM users WHERE';
+ $prepare_params = [];
+  
+ if (!empty($_GET['name'])) {
+   $sql .= ' name LIKE :name';
+   $prepare_params[':name'] = '%' . $_GET['name'] . '%';
+ }
+ 
+ if (empty($prepare_params)) {
+   // 検索がまったくない場合はWHERE句が空になってしまうため真となる適当な式を設定する
+   $sql .= ' 1 = 1';
+ }
+ 
+ $sql .= ' ORDER BY id DESC';
+ 
+ $select_sth = $dbh->prepare($sql);
+ $select_sth->execute($prepare_params);

// ログインしている場合、フォローしている会員IDリストを取得
$followee_user_ids = [];
```

```diff
    <a href="/timeline.php">タイムライン</a>
  </div>

+   <div style="margin-bottom: 1em;">
+     絞り込み<br>
+     <form method="GET">
+       名前: <input type="text" name="name" value="<?= htmlspecialchars($_GET['name'] ?? '') ?>"><br>
+       <button type="submit">決定</button>
+     </form>
+   </div>
+ 
  <?php foreach($select_sth as $user): ?>
    <div style="display: flex; justify-content: start; align-items: center; padding: 1em 2em;">
      <?php if(empty($user['icon_filename'])): ?>
```

8. 生まれ年で検索
vim public/users.php
```diff
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');

// 会員データを取得
- $sql = 'SELECT * FROM users WHERE';
+ $sql = 'SELECT * FROM users';
+ $where_sql_array = [];
$prepare_params = [];

if (!empty($_GET['name'])) {
-   $sql .= ' name LIKE :name';
+   $where_sql_array[] = ' name LIKE :name';
  $prepare_params[':name'] = '%' . $_GET['name'] . '%';
}
+ if (!empty($_GET['year_from'])) {
+   $where_sql_array[] = ' birthday >= :year_from';
+   $prepare_params[':year_from'] = $_GET['year_from'] . '-01-01'; // 入力年の1月1日
+ }
+ if (!empty($_GET['year_until'])) {
+   $where_sql_array[] = ' birthday <= :year_until';
+   $prepare_params[':year_until'] = $_GET['year_until'] . '-12-31'; // 入力年の12月31日
+ }

- if (empty($prepare_params)) {
-   // 検索がまったくない場合はWHERE句が空になってしまうため真となる適当な式を設定する
-   $sql .= ' 1 = 1';
+ if (!empty($where_sql_array)) {
+   $sql .= ' WHERE ' . implode(' AND', $where_sql_array);
}

$sql .= ' ORDER BY id DESC';
```

```diff
    絞り込み<br>
    <form method="GET">
      名前: <input type="text" name="name" value="<?= htmlspecialchars($_GET['name'] ?? '') ?>"><br>
+       生まれ年:
+       <input type="number" name="year_from" value="<?= htmlspecialchars($_GET['year_from'] ?? '') ?>">年
+       ~
+       <input type="number" name="year_until" value="<?= htmlspecialchars($_GET['year_until'] ?? '') ?>">年
+       <br>
      <button type="submit">決定</button>
    </form>
  </div>
```

```diff
      <a href="/profile.php?user_id=<?= $user['id'] ?>" style="margin-left: 1em;">
        <?= htmlspecialchars($user['name']) ?>
      </a>
+ 
      <div style="margin-left: 2em;">
        <?php if($user['id'] === $_SESSION['login_user_id']): ?>
          これはあなたです!
```
タイムラインをJSでレンダリングする
1. JSON形式でタイムラインの情報を出力する
vim public/timeline_json.php
```
<?php
$dbh = new PDO('mysql:host=mysql;dbname=techc', 'root', '');

session_start();
if (empty($_SESSION['login_user_id'])) { // 非ログインの場合利用不可 401 で空のものを返す
  header("HTTP/1.1 401 Unauthorized");
  header("Content-Type: application/json");
  print(json_encode(['entries' => []]));
  return;
}

// 現在のログイン情報を取得する
$user_select_sth = $dbh->prepare("SELECT * from users WHERE id = :id");
$user_select_sth->execute([':id' => $_SESSION['login_user_id']]);
$user = $user_select_sth->fetch();

// 投稿データを取得。IN句の中身もプレースホルダを使うために、$target_user_ids の要素数だけ「?」を付けている。
$sql = 'SELECT bbs_entries.*, users.name AS user_name, users.icon_filename AS user_icon_filename'
  . ' FROM bbs_entries'
  . ' INNER JOIN users ON bbs_entries.user_id = users.id'
  . ' WHERE'
  . '   bbs_entries.user_id IN'
  . '     (SELECT followee_user_id FROM user_relationships WHERE follower_user_id = :login_user_id)'
  . '   OR bbs_entries.user_id = :login_user_id'
  . ' ORDER BY bbs_entries.created_at DESC';
$select_sth = $dbh->prepare($sql);
$select_sth->execute([
  ':login_user_id' => $_SESSION['login_user_id'],
]);

// bodyのHTMLを出力するための関数を用意する
function bodyFilter (string $body): string
{
  $body = htmlspecialchars($body); // エスケープ処理
  $body = nl2br($body); // 改行文字を<br>要素に変換

  // >>1 といった文字列を該当番号の投稿へのページ内リンクとする (レスアンカー機能)
  // 「>」(半角の大なり記号)は htmlspecialchars() でエスケープされているため注意
  $body = preg_replace('/&gt;&gt;(\d+)/', '<a href="#entry$1">&gt;&gt;$1</a>', $body);

  return $body;
}

// JSONに吐き出す用のentries
$result_entries = [];
foreach ($select_sth as $entry) {
  $result_entry = [
    'id' => $entry['id'],
    'user_name' => $entry['user_name'],
    'user_profile_url' => '/profile.php?user_id=' . $entry['user_id'],
    'body' => bodyFilter($entry['body']),
    'created_at' => $entry['created_at'],
  ];
  $result_entries[] = $result_entry;
}

header("HTTP/1.1 200 OK");
header("Content-Type: application/json");
print(json_encode(['entries' => $result_entries]));
```

2. JSON形式の情報をもとにJSでHTMLをレンダリングする
vim public/timeline.php
```diff
  header("Location: ./timeline.php");
  return;
}

- // 投稿データを取得。
- // フォローしている人の投稿と自分自身の投稿のみ表示
- $sql = 'SELECT bbs_entries.*, users.name AS user_name, users.icon_filename AS user_icon_filename'
-   . ' FROM bbs_entries'
-   . ' INNER JOIN users ON bbs_entries.user_id = users.id'
-   . ' LEFT OUTER JOIN user_relationships ON bbs_entries.user_id = user_relationships.followee_user_id'
-   . ' WHERE user_relationships.follower_user_id = :login_user_id OR bbs_entries.user_id = :login_user_id'
-   . ' ORDER BY bbs_entries.created_at DESC';
- $select_sth = $dbh->prepare($sql);
- $select_sth->execute([
-     ':login_user_id' => $_SESSION['login_user_id'],
- ]);
- 
- // bodyのHTMLを出力するための関数を用意する
- function bodyFilter (string $body): string
- {
-     $body = htmlspecialchars($body); // エスケープ処理
-     $body = nl2br($body); // 改行文字を<br>要素に変換
- 
-     // >>1 といった文字列を該当番号の投稿へのページ内リンクとする (レスアンカー機能)
-     // 「>」(半角の大なり記号)は htmlspecialchars() でエスケープされているため注意
-      $body = preg_replace('/&gt;&gt;(\d+)/', '<a href="#entry$1">&gt;&gt;$1</a>', $body);
- 
-     return $body;
- }
?>

<div>
```

```diff
  <canvas id="imageCanvas" style="display: none;"></canvas><!-- 画像縮小に使うcanvas (非表示) -->
  <button type="submit">送信</button>
</form>

- <?php foreach($select_sth as $entry): ?>
-   <dl style="margin-bottom: 1em; padding-bottom: 1em; border-bottom: 1px solid #ccc;">
-     <dt id="entry<?= htmlspecialchars($entry['id']) ?>">
-       番号
-     </dt>
-     <dd>
-       <?= htmlspecialchars($entry['id']) ?>
-     </dd>
-     <dt>
-       投稿者
-     </dt>
-     <dd>
-       <a href="/profile.php?user_id=<?= $entry['user_id'] ?>">
-         <?php if(!empty($entry['user_icon_filename'])): // アイコン画像がある場合は表示 ?>
-         <img src="/image/<?= $entry['user_icon_filename'] ?>"
-           style="height: 2em; width: 2em; border-radius: 50%; object-fit: cover;">
-         <?php endif; ?>
- 
-         <?= htmlspecialchars($entry['user_name']) ?>
-         (ID: <?= htmlspecialchars($entry['user_id']) ?>)
-       </a>
-     </dd>
-     <dt>日時</dt>
-     <dd><?= $entry['created_at'] ?></dd>
-     <dt>内容</dt>
-     <dd>
-       <?= bodyFilter($entry['body']) ?>
-       <?php if(!empty($entry['image_filename'])): ?>
-       <div>
-         <img src="/image/<?= $entry['image_filename'] ?>" style="max-height: 10em;">
-       </div>
-       <?php endif; ?>
-     </dd>
-   </dl>
- <?php endforeach ?>
+ <hr>
+ 
+ <dl id="entryTemplate" style="display: none; margin-bottom: 1em; padding-bottom: 1em; border-bottom: 1px solid #ccc;">
+   <dt>番号</dt>
+   <dd data-role="entryIdArea"></dd>
+   <dt>投稿者</dt>
+   <dd>
+     <a href="" data-role="entryUserAnchor"></a>
+   </dd>
+   <dt>日時</dt>
+   <dd data-role="entryCreatedAtArea"></dd>
+   <dt>内容</dt>
+   <dd data-role="entryBodyArea">
+   </dd>
+ </dl>
+ <div id="entriesRenderArea"></div>

<script>
document.addEventListener("DOMContentLoaded", () => {
+   const entryTemplate = document.getElementById('entryTemplate');
+   const entriesRenderArea = document.getElementById('entriesRenderArea');
+ 
+   const request = new XMLHttpRequest();
+   request.onload = (event) => {
+     const response = event.target.response;
+     response.entries.forEach((entry) => {
+       // テンプレートとするものから要素をコピー
+       const entryCopied = entryTemplate.cloneNode(true);
+ 
+       // display: none を display: block に書き換える
+       entryCopied.style.display = 'block';
+ 
+       // id属性を設定しておく(レスアンカ用)
+       entryCopied.id = 'entry' + entry.id.toString();
+ 
+       // 番号(ID)を表示
+       entryCopied.querySelector('[data-role="entryIdArea"]').innerText = entry.id.toString();
+ 
+       // 名前を表示
+       entryCopied.querySelector('[data-role="entryUserAnchor"]').innerText = entry.user_name;
+ 
+       // 名前のところのリンク先(プロフィール)のURLを設定
+       entryCopied.querySelector('[data-role="entryUserAnchor"]').href = entry.user_profile_url;
+ 
+       // 投稿日時を表示
+       entryCopied.querySelector('[data-role="entryCreatedAtArea"]').innerText = entry.created_at;
+ 
+       // 本文を表示 (ここはHTMLなのでinnerHTMLで)
+       entryCopied.querySelector('[data-role="entryBodyArea"]').innerHTML = entry.body;
+ 
+       // 最後に実際の描画を行う
+       entriesRenderArea.appendChild(entryCopied);
+     });
+   }
+   request.open('GET', '/timeline_json.php', true); // timeline_json.php を叩く
+   request.responseType = 'json';
+   request.send();
+ 
+ 
+   // 以下画像縮小用
  const imageInput = document.getElementById("imageInput");
  imageInput.addEventListener("change", () => {
    if (imageInput.files.length < 1) {
```

3. 投稿の画像も表示されるように
vim public/timeline.php
```diff
      // 本文を表示 (ここはHTMLなのでinnerHTMLで)
      entryCopied.querySelector('[data-role="entryBodyArea"]').innerHTML = entry.body;

+       // 画像が存在する場合に本文の下部に画像を表示
+       if (entry.image_file_url !== undefined) {
+         const imageElement = new Image();
+         imageElement.src = entry.image_file_url; // 画像URLを設定
+         imageElement.style.display = 'block'; // ブロック要素にする (img要素はデフォルトではインライン要素のため)
+         imageElement.style.marginTop = '1em'; // 画像上部の余白を設定
+         imageElement.style.maxHeight = '300px'; // 画像を表示する最大サイズ(縦)を設定
+         imageElement.style.maxWidth = '300px'; // 画像を表示する最大サイズ(横)を設定
+         entryCopied.querySelector('[data-role="entryBodyArea"]').appendChild(imageElement); // 本文エリアに画像を追加
+       }
+ 
      // 最後に実際の描画を行う
      entriesRenderArea.appendChild(entryCopied);
    });
```

vim public/timeline_json.php
```diff
    'user_name' => $entry['user_name'],
    'user_profile_url' => '/profile.php?user_id=' . $entry['user_id'],
    'body' => bodyFilter($entry['body']),
+     'image_file_url' => empty($entry['image_filename']) ? '' : ('/image/' . $entry['image_filename']),
    'created_at' => $entry['created_at'],
  ];
  $result_entries[] = $result_entry;
```

4. 投稿者のアイコン画像も表示
vim public/timeline.php
```diff
  <dd data-role="entryIdArea"></dd>
  <dt>投稿者</dt>
  <dd>
-     <a href="" data-role="entryUserAnchor"></a>
+     <a href="" data-role="entryUserAnchor">
+       <img data-role="entryUserIconImage"
+         style="height: 2em; width: 2em; border-radius: 50%; object-fit: cover;">
+       <span data-role="entryUserNameArea"></span>
+     </a>
  </dd>
  <dt>日時</dt>
  <dd data-role="entryCreatedAtArea"></dd>
```

```diff
      // 番号(ID)を表示
      entryCopied.querySelector('[data-role="entryIdArea"]').innerText = entry.id.toString();

+       // アイコン画像が存在する場合は表示 なければimg要素ごと非表示に
+       if (entry.user_icon_file_url !== undefined) {
+         entryCopied.querySelector('[data-role="entryUserIconImage"]').src = entry.user_icon_file_url;
+       } else {
+         entryCopied.querySelector('[data-role="entryUserIconImage"]').display = 'none';
+       }
+ 
      // 名前を表示
-       entryCopied.querySelector('[data-role="entryUserAnchor"]').innerText = entry.user_name;
+       entryCopied.querySelector('[data-role="entryUserNameArea"]').innerText = entry.user_name;

      // 名前のところのリンク先(プロフィール)のURLを設定
      entryCopied.querySelector('[data-role="entryUserAnchor"]').href = entry.user_profile_url;
```

vim public/timeline_json.php
```diff
  $result_entry = [
    'id' => $entry['id'],
    'user_name' => $entry['user_name'],
+     'user_icon_file_url' => empty($entry['user_icon_filename']) ? '' : ('/image/' . $entry['user_icon_filename']),
    'user_profile_url' => '/profile.php?user_id=' . $entry['user_id'],
    'body' => bodyFilter($entry['body']),
    'image_file_url' => empty($entry['image_filename']) ? '' : ('/image/' . $entry['image_filename']),
```
