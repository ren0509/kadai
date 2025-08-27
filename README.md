１ssh ec2-user@54.90.120.177 -i C:\Users\ktc\Downloads\kadailey.pem 

２sudo yum install vim -y 

３vim ~/.vimrc 

set number 

set expandtab 

set tabstop=2 

set shiftwidth=2 

set autoindent 

4sudo yum install screen -y 

5screen 

vim ~/.screenrc 

hardstatus alwayslastline "%{= bw}%-w%{= wk}%n%t*%{-}%+w" 

6 sudo yum install -y docker 

sudo systemctl start docker 

sudo systemctl enable docker 

sudo usermod -a -G docker ec2-user 

7 sudo mkdir -p /usr/local/lib/docker/cli-plugins/ 

sudo curl -SL https://github.com/docker/compose/releases/download/v2.36.0/docker-compose-linux-x86_64 -o /usr/local/lib/docker/cli-plugins/docker-compose 

sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose 

　docker compose version 

8 mkdir dockertest 

cd dockertest 

9 vim compose.yml 

　services: 

  web: 

    image: nginx:latest 

    ports: 

      - 80:80 

    volumes: 

      - ./nginx/conf.d/:/etc/nginx/conf.d/ 

      - ./public/:/var/www/public/ 

      - image:/var/www/upload/image/ 

    depends_on: 

      - php 

  php: 

    container_name: php 

    build: 

      context: . 

      target: php 

    volumes: 

      - ./public/:/var/www/public/ 

      - image:/var/www/upload/image/ 

  mysql: 

    container_name: mysql 

    image: mysql:8.4 

    environment: 

      MYSQL_DATABASE: example_db 

      MYSQL_ALLOW_EMPTY_PASSWORD: 1 

      TZ: Asia/Tokyo 

    volumes: 

      - mysql:/var/lib/mysql 

    command: > 

      mysqld 

      --character-set-server=utf8mb4 

      --collation-server=utf8mb4_unicode_ci 

      --max_allowed_packet=4MB 

volumes: 

  mysql: 

  image: 

10 docker compose up 

11 mkdir nginx 

mkdir nginx/conf.d 

12 vim nginx/conf.d/default.conf 

  

server { 

    listen       0.0.0.0:80; 

    server_name  _; 

    charset      utf-8; 

 

    root /var/www/public; 

 

    location ~ \.php$ { 

      fastcgi_pass php:9000; 

      	fastcgi_index index.php; 

      fastcgi_param SCRIPT_FILENAME  $document_root$fastcgi_script_name; 

      	include       fastcgi_params; 

    } 

 

    location /image/ { 

        root /var/www/upload; 

    } 

                 

} 

13 mkdir public 

14 vim public/kadai.php 

<?php 

$dbh = new PDO('mysql:host=mysql;dbname=example_db', 'root', ''); 

 

if (isset($_POST['body'])) { 

  // POSTで送られてくるフォームパラメータ body がある場合 

 

  $image_filename = null; 

  if (isset($_FILES['image']) && !empty($_FILES['image']['tmp_name'])) { 

    // アップロードされた画像がある場合 

    if (preg_match('/^image\//', mime_content_type($_FILES['image']['tmp_name'])) !== 1) { 

      // アップロードされたものが画像ではなかった場合処理を強制的に終了 

      header("HTTP/1.1 302 Found"); 

      header("Location: ./bbsimagetest.php"); 

      return; 

    } 

 

    // 元のファイル名から拡張子を取得 

    $pathinfo = pathinfo($_FILES['image']['name']); 

    $extension = $pathinfo['extension']; 

    // 新しいファイル名を決める。他の投稿の画像ファイルと重複しないように時間+乱数で決める。 

    $image_filename = strval(time()) . bin2hex(random_bytes(25)) . '.' . $extension; 

    $filepath =  '/var/www/upload/image/' . $image_filename; 

    move_uploaded_file($_FILES['image']['tmp_name'], $filepath); 

  } 

 

  // insertする 

  $insert_sth = $dbh->prepare("INSERT INTO bbs_entries (body, image_filename) VALUES (:body, :image_filename)"); 

  $insert_sth->execute([ 

    ':body' => $_POST['body'], 

    ':image_filename' => $image_filename, 

  ]); 

 

  // 処理が終わったらリダイレクトする 

  // リダイレクトしないと，リロード時にまた同じ内容でPOSTすることになる 

  header("HTTP/1.1 302 Found"); 

  header("Location: ./bbsimagetest.php"); 

  return; 

} 

 

// いままで保存してきたものを取得 

$select_sth = $dbh->prepare('SELECT * FROM bbs_entries ORDER BY created_at DESC'); 

$select_sth->execute(); 

?> 

 

<!-- フォームのPOST先はこのファイル自身にする --> 

<form method="POST" action="./bbsimagetest.php" enctype="multipart/form-data"> 

  <textarea name="body" required></textarea> 

  <div style="margin: 1em 0;"> 

    <input type="file" accept="image/*" name="image" id="imageInput"> 

  </div> 

  <button type="submit">送信</button> 

</form> 

 

<hr> 

 

<?php foreach($select_sth as $entry): ?> 

  <dl style="margin-bottom: 1em; padding-bottom: 1em; border-bottom: 1px solid #ccc;"> 

    <dt>ID</dt> 

    <dd><?= $entry['id'] ?></dd> 

    <dt>日時</dt> 

    <dd><?= $entry['created_at'] ?></dd> 

    <dt>内容</dt> 

    <dd> 

      <?= nl2br(htmlspecialchars($entry['body'])) // 必ず htmlspecialchars() すること ?> 

      <?php if(!empty($entry['image_filename'])): // 画像がある場合は img 要素を使って表示 ?> 

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

    if (imageInput.files[0].size > 5 * 1024 * 1024) { 

      // ファイルが5MBより多い場合 

      alert("5MB以下のファイルを選択してください。"); 

      imageInput.value = ""; 

    } 

  }); 

}); 

</script> 

 

 

15 vim Dockerfile 

FROM php:8.4-fpm-alpine AS php 

  

RUN docker-php-ext-install pdo_mysql 

  

RUN install -o www-data -g www-data -d /var/www/upload/image/ 

  

RUN docker-php-ext-install fileinfo && docker-php-ext-enable fileinfo 

  

COPY uploads.ini /usr/local/etc/php/conf.d/uploads.ini 

 16 vim uploads.ini  

  ; ファイルアップロードの上限を設定 

upload_max_filesize = 5M 

  

; POSTリクエスト全体のデータサイズ上限。ファイルサイズより少し大きくする 

post_max_size = 6M 

 

17 dockercompose up後に
docker compose exec mysql mysql example_db 
 
CREATE TABLE `example_db`.`bbs_entries` ( 
  `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY, 
  `body` TEXT NOT NULL, 
  `image_filename` VARCHAR(255), 
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP 
 );
