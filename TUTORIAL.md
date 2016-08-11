# Setup, Build and Run a Rails 5 web application with Docker and Docker Compose

## Setup the project

In order to kickstart the application we need to install the rails gem and run the
`rails new <appname>` command. We will do this inside the same Docker container
that will be used to run the application itself. To do so we need to create a
`Dockerfile` to create the Docker image with the necessary dependencies, a
`docker-compose.yml` file to provision the other services needed (a postgres
database and a volume store) and a `Gemfile` (with an empty `Gemfile.lock`) to
install the `rails` gem and bundle install its dependencies.

Create a new directory and some empty files:

```
$ mkdir rails-docker
$ cd rails-docker
$ touch Dockerfile docker-compose.yml Gemfile Gemfile.lock
```

Copy and paste the following content in the respective files:

File: `Dockerfile`
```
FROM ruby:2.3.1

ENV APP_ROOT /app
ENV BUNDLE_PATH /bundle

RUN apt-get update -qq && \
    apt-get install -y build-essential libpq-dev nodejs

WORKDIR $APP_ROOT
ADD . $APP_ROOT
```

File: `docker-compose.yml`
```
version: '2'
services:
  store:
    # data-only container
    image: postgres:latest # reuse postgres container
    volumes:
      - /var/lib/postgresql/data
    command: "true"
  db:
    image: postgres
    ports:
      - 5432:5432
    volumes_from:
      - store # connect postgres and the data-only container
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=rails_docker_database
  web:
    build: .
    ports:
      - 3000:3000
    volumes:
      - .:/app
    # This tells the web container to mount the `bundle` images'
    # /bundle volume to the `web` containers /bundle path.
    volumes_from:
      - bundle
    links:
      - db
    command: ./bin/start.sh
    environment:
      - PORT=3000
      - DB_HOST=db
      - DB_PORT=5432
      - DB_NAME=rails_docker_database
      - DB_USER=postgres
      - DB_PSWD=postgres
  bundle:
    image: busybox
    volumes:
      - /bundle
```

File: `Gemfile`
```
source 'https://rubygems.org'

gem 'rails', '~> 5'
```

## Build the project

First we need to bundle the rails 5 dependencies:
```
$ docker-compose run web bundle
```

And then use the `rails new` command to create the new application:
```
$ docker-compose run web bundle exec rails new . --force --database=postgresql --skip-bundle
```

## Configure the database

We need to change slightly the database configuration to use the environment
variables set in the docker-compose file:

File: `config/database.yml`
```
default: &default
  adapter: postgresql
  encoding: unicode
  username: <%= ENV['DB_USER'] %>
  password: <%= ENV['DB_PSWD'] %>
  host: <%= ENV['DB_HOST'] %>
  # For details on connection pooling, see rails configuration guide
  # http://guides.rubyonrails.org/configuring.html#database-pooling
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: <%= ENV['DB_NAME'] %>

test:
  <<: *default
  database: app_test
```

## Setup the app

```
$ docker-compose run web bin/setup
```

Create a `start.sh` file in the `bin` dir:

File: `bin/start.sh`
```
#!/bin/bash

bundle check || bundle install

if [ -f tmp/pids/server.pid ]; then
  rm -f tmp/pids/server.pid
fi

bundle exec rails s -p $PORT -b 0.0.0.0
```

It automatically removes the `server.pid` that will create problems when you stop
and restart the app.
Make the file executable with the `chmod` command:
```
$ chmod +x bin/start.sh
```

Finally start your newly created Rails application and visit [http://localhost:3000]:
```
$ docker-compose up
```
