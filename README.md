# myApp
フロントエンド：Astro、バックエンド：Laravelの開発環境

WSL (Ubuntu) 上で、Astro + Laravel + MySQL + GitHub を使った開発環境をDockerで構築する手順についてですね。

この構成は、フロントエンドとバックエンドが独立している現代的なWeb開発の典型例です。DockerとDocker Composeを使うことで、各コンポーネントを独立したコンテナとして管理し、環境構築を効率化できます。

ここでは、`docker-compose.yml` を中心とした構築手順を解説します。

### 全体のディレクトリ構成

まず、プロジェクトのルートディレクトリを作成し、以下のような構成を想定します。

```
/your-project
├── docker-compose.yml
├── .env
├── .git
├── frontend/             # Astroプロジェクトのディレクトリ
│   ├── src/
│   ├── public/
│   ├── package.json
│   └── ...
├── backend/              # Laravelプロジェクトのディレクトリ
│   ├── app/
│   ├── public/
│   ├── bootstrap/
│   ├── vendor/
│   ├── artisan
│   └── ...
└── mysql/                # MySQLの設定や初期データ（任意）
    └── my.cnf
```

### ステップ1: Docker DesktopのインストールとWSL2の設定

まだインストールしていない場合は、Docker Desktop for Windowsをインストールしてください。

1.  **Docker Desktop for Windowsをインストール**：
      - [Docker公式ウェブサイト](https://www.docker.com/products/docker-desktop/)からダウンロードし、インストールします。
      - インストール時に「WSL 2 based engineを使用する」オプションが有効になっていることを確認します。
2.  **WSL2を有効化**：
      - PowerShellを管理者として開き、`wsl --install`コマンドを実行します。
      - `wsl --list --verbose`でWSL2が有効になっているか確認し、必要であれば`wsl --set-version <ディストリビューション名> 2`でWSL2にアップグレードします。
3.  **Docker Desktopの設定**：
      - Docker Desktopを起動し、**Settings \> Resources \> WSL Integration** に移動します。
      - 使用するUbuntuディストリビューションが有効になっていることを確認します。

### ステップ2: `docker-compose.yml`の作成

このファイルが、すべてのコンテナ（サービス）を定義する核となります。

プロジェクトのルートディレクトリに`docker-compose.yml`というファイルを作成し、以下のように記述します。

```yaml
version: '3.8'

services:
  # フロントエンド（Astro）
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: astro_frontend
    volumes:
      - ./frontend:/app
    ports:
      - "4321:4321" # Astroの開発サーバーポート
    command: npm run dev
    tty: true

  # バックエンド（Laravel + PHP-FPM）
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: laravel_backend
    volumes:
      - ./backend:/var/www/html
    tty: true
    depends_on:
      - db
      - webserver

  # Webサーバー（Nginx）
  webserver:
    image: nginx:latest
    container_name: nginx_server
    ports:
      - "8080:80" # ブラウザからアクセスするポート
    volumes:
      - ./backend:/var/www/html
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - backend

  # データベース（MySQL）
  db:
    image: mysql:8.0
    container_name: mysql_db
    ports:
      - "3306:3306"
    volumes:
      - dbdata:/var/lib/mysql
      - ./docker/mysql/my.cnf:/etc/mysql/conf.d/my.cnf
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_DATABASE}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}

volumes:
  dbdata:
```

### ステップ3: Dockerfileと設定ファイルの作成

各コンポーネントに対応する`Dockerfile`と設定ファイルを作成します。

#### 3-1. フロントエンド (`frontend/Dockerfile`)

Astroの開発に必要なNode.js環境を構築します。

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

# ホストPCからマウントするため、COPYは不要
# COPY . .

EXPOSE 4321

CMD ["npm", "run", "dev"]
```

#### 3-2. バックエンド (`backend/Dockerfile`)

PHP 8.2とComposer、およびLaravelプロジェクトに必要な拡張機能をインストールします。

```dockerfile
FROM php:8.2-fpm-alpine

# 必要な拡張機能のインストール
RUN apk add --no-cache \
    $PHPIZE_DEPS \
    libzip-dev \
    libpng-dev \
    jpeg-dev \
    git \
    oniguruma-dev \
    libxml2-dev

RUN docker-php-ext-install pdo pdo_mysql zip exif pcntl gd

# Composerのインストール
COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer

WORKDIR /var/www/html

COPY . .

# Composerの依存関係をインストール
# `docker-compose up`時に実行する場合はコメントアウト
# RUN composer install --no-dev --optimize-autoloader

EXPOSE 9000

CMD ["php-fpm"]
```

#### 3-3. Nginx設定ファイル (`docker/nginx/default.conf`)

NginxがPHP-FPMにリクエストをプロキシするように設定します。

```nginx
server {
    listen 80;
    server_name localhost;
    root /var/www/html/public;

    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass backend:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

### ステップ4: `.env`ファイルの作成

`.env`ファイルを作成し、データベースの接続情報を定義します。

```
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=your_db
DB_USERNAME=your_user
DB_PASSWORD=your_password

DB_ROOT_PASSWORD=your_root_password
```

### ステップ5: プロジェクトの初期化とコンテナの起動

1.  **Astroプロジェクトの作成**：
      - `frontend`ディレクトリに移動し、`npm create astro@latest .`でプロジェクトを作成します。
2.  **Laravelプロジェクトの作成**：
      - `backend`ディレクトリに移動し、`composer create-project laravel/laravel .`でプロジェクトを作成します。
3.  **Dockerコンテナの起動**：
      - プロジェクトのルートディレクトリに戻り、以下のコマンドを実行します。
    <!-- end list -->
    ```bash
    docker compose up -d --build
    ```
      - 初回はイメージのビルドに時間がかかります。
4.  **Laravelの依存関係インストール**：
      - `backend`コンテナに入り、Composerの依存関係をインストールします。
    <!-- end list -->
    ```bash
    docker compose exec backend composer install
    ```
5.  **Laravelの設定**：
      - `.env`ファイルが作成されているか確認し、DB情報を更新します。
      - アプリケーションキーを生成します。
    <!-- end list -->
    ```bash
    docker compose exec backend php artisan key:generate
    ```
6.  **マイグレーションとシーダーの実行**：
      - データベースのテーブルを作成します。
    <!-- end list -->
    ```bash
    docker compose exec backend php artisan migrate
    ```

### 完成！

  - **Astro**：`http://localhost:4321` にアクセスして確認します。
  - **Laravel**：`http://localhost:8080` にアクセスして確認します。

これで、WSL上のDockerでAstroとLaravelが連携する開発環境が完成です。この環境をGitHubで管理すれば、チームメンバーも`git clone`後に`docker compose up`だけで同じ環境を簡単に構築できます。
