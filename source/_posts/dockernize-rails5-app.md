---
title: Rails5の開発環境をDockernizeする(MySQL + Spring + Sidekiq + Redis + docker-sync)
date: 2017-12-31 15:24:30
categories:
  - development
author: me
layout: blog
---

年の瀬など関係なく過ごしております。

既存のRailsアプリの開発環境をDocker化する機会がありましたので、やったことを記録しておきます。

とは言っても、先人たちの知恵がかなり参考になりましたので、それを組み合わせて調整した程度の焼き直しです。

# 前提
- `docker-compose up` 一発で開発環境構築が済むようにしたい
- 本番環境のコンテナ化は別のDockerfile & docker-compose.ymlで行う

# 参考にしたページ（一部）
- [高速に開発できる Docker + Rails開発環境のテンプレートを作った](https://qiita.com/kawasin73/items/2253523be18e5afd994f)
	- 上記ページ内から以下2つの記事に言及されています
	- [開発しやすいRails on Docker環境の作り方](https://qiita.com/joker1007/items/9f54e763ae640f757cfb)
	- [Railsアプリケーション開発を完全にDocker化する](http://tech.degica.com/ja/2016/06/14/dockerized-rails-development/)

とても参考になりました。ありがとうございます。

# 成果物

## Dockerfile.dev

```
FROM ruby:2.4.2

ENV LANG C.UTF-8

RUN apt-get update -qq
RUN apt-get install -y --no-install-recommends \
  build-essential \
  curl \
  wget \
  apt-transport-https \
  git \
  libpq-dev \
  libfontconfig1 && \
  rm -rf /var/lib/apt/lists/*

# install node
RUN curl -sL https://deb.nodesource.com/setup_6.x | bash -
RUN apt-get install -y nodejs

# install yarn
RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
RUN echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list
RUN apt-get update
RUN apt-get install -y yarn

ENV ENTRYKIT_VERSION 0.4.0

RUN wget https://github.com/progrium/entrykit/releases/download/v${ENTRYKIT_VERSION}/entrykit_${ENTRYKIT_VERSION}_Linux_x86_64.tgz \
  && tar -xvzf entrykit_${ENTRYKIT_VERSION}_Linux_x86_64.tgz \
  && rm entrykit_${ENTRYKIT_VERSION}_Linux_x86_64.tgz \
  && mv entrykit /bin/entrykit \
  && chmod +x /bin/entrykit \
  && entrykit --symlink

RUN mkdir /app

WORKDIR /app

RUN bundle config build.nokogiri --use-system-libraries

ENTRYPOINT [ \
  "prehook", "ruby -v", "--", \
  "prehook", "bundle install -j3 --quiet", "--", \
  "prehook", "yarn install", "--"]
```

ポイント：

- nodeのインストール
	- railsアプリでuglifierなどの[ExecJS](https://github.com/sstephenson/execjs)が必要なgemをいくつか使っていますので、予めnodeをインストールしておきます
- yarnのインストール
	- こちらも同様に、railsアプリ内で利用するnpmモジュールを `package.json` で管理しておりますので、予めyarnをインストールしておきます
- ENTRYKITの活用
	- 参考サイトそのままですが、ENTRYKITを活用することで、起動時に必要な処理（ここでは `bundle install` や `yarn install`）などを完結に記述できます

## config/

## docker-compose.yml

```
version: '2'
services:
  rails: &app_base
    build:
      context: .
      dockerfile: "Dockerfile.dev"
    command: ["bundle", "exec", "rails", "s"]
    env_file:
      - "./.env.dev"
    volumes:
      - "rails-sync:/app"
    volumes_from:
      - data
    ports:
      - "3000:3000"
      - "9292:9292"
    depends_on:
      - db
    tty: true
    stdin_open: true
  worker:
    <<: *app_base
    command: ["bundle", "exec", "sidekiq"]
    ports: []
    depends_on:
      - db
      - redis
  db:
    image: "mysql:5.7.20"
    environment:
      MYSQL_ROOT_PASSWORD: password
    volumes_from:
      - data
    ports:
      - "3306:3306"
  redis:
    image: redis:alpine
    networks:
      - default
    ports:
      - '6379:6379'
    volumes_from:
      - data
  data:
    image: busybox
    volumes:
      - "db:/var/lib/mysql"
      - "bundle:/usr/local/bundle"
      - "redis:/data/redis"
      - "node_modules:/app/node_modules"

volumes:
  db:
    driver: local
  bundle:
    driver: local
  redis:
    driver: local
  node_modules:
    driver: local
  rails-sync:
    external: true
```

ポイント：

- railsコンテナのport 9292
	- `config/puma.rb` 内でSSL with オレオレ証明書を有効にしているので、ポートを空けています

		```ruby
		if "development" == ENV.fetch("RAILS_ENV") { "development" }
		  ssl_bind '0.0.0.0', '9292', {
		    key: 'dev-cert/server.key',
		    cert: 'dev-cert/server.crt',
		    verify_mode: 'none'
		  }
		```
- 各種データストア, bundle, node_modulesの永続化
	- busybox	imageを使って、永続化が必要なデータを隔離しています
	- 新しいモジュールを追加しても、bundle installやyarn installの実行は短くて済みます
- springコンテナは不要だった
	- 上記参考サイトにはspringコンテナが別途必要とのことだったのですが`bin/rails`や`bin/rake` 内にspringを読み込むコードがあれば、`docker-compose exec rails rails c`でSpring preloader経由で呼び出されました
	- なぜだろう
- worker / sidekiqコンテナ
	- sidekiqを起動するために、別でコンテナを定義しています
- 今だったら、`version: 3`の記法で書いてもよかったかもしれない

## docker-syncについて

docker-sync.yml

```
version: "2"

options:
  verbose: true
syncs:
  rails-sync: # tip: add -sync and you keep consistent names as a convention
    src: '.'
    # sync_strategy: 'native_osx' # not needed, this is the default now
    # sync_excludes: ['ignored_folder', '.ignored_dot_folder']
```

Docker for Macにはファイルシステムのパフォーマンス上の[問題](https://github.com/docker/for-mac/issues/77)があるため、workaroundとして、次の2種類が有力です。

1. `cached`オプションの使用 [参考](https://github.com/docker/for-mac/issues/1592)
  - volumeマウント時にどのレベルでファイルの一貫性を要求するか、というオプションが設定されました
  - 設定して試してみましたが私の環境では効果がなかったので、docker-syncを採用しました
2. docker-sync
  - 上記の通り、docker-sync.ymlという設定ファイルに`rails-sync`という名前でdocker-sync用のvolumeを作成します
  - railsコンテナからは、`rails-sync` volumeをマウントします

  ```
      volumes:
      - "rails-sync:/app"
  ```
    - 今回は、開発環境のみでしかdocker-compose.ymlを使用していませんが、もしdocker-composeが本番用にもある場合は[こちら](https://github.com/EugenMayer/docker-sync/wiki/Keep-your-docker-compose.yml-portable)のテクニックが参考になります
    - docker-syncの設定ファイルには多数のオプションがありますので、気になる方は[公式ページ](https://github.com/EugenMayer/docker-sync/wiki/2.-Configuration)を確認するとよいでしょう。
    - 手元の環境では、docker-syncを使用することでページ表示までの時間が約3倍ほど高速化（2s -> 0.7s）されました。よかったですね。

# まとめ
こういう作業をすべてゼロからやっていたと思うと恐ろしいですが、先人の知恵のおかげで1日で快適な開発環境を手に入れることができました。

みなさまよいお年を。
