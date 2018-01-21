# Heroku Buildpack: NGINX

NGINX-buildpack vendors NGINX inside a dyno and connects NGINX to an app server via UNIX domain sockets.

## Motivation

Some application servers (e.g. Ruby's Unicorn) halt progress when dealing with network I/O. Heroku's Cedar routing stack [buffers only the headers](https://devcenter.heroku.com/articles/http-routing#request-buffering) of inbound requests. (The Cedar router will buffer the headers and body of a response up to 1MB) Thus, the Heroku router engages the dyno during the entire body transfer –from the client to dyno. For applications servers with blocking I/O, the latency per request will be degraded by the content transfer. By using NGINX in front of the application server, we can eliminate a great deal of transfer time from the application server. In addition to making request body transfers more efficient, all other I/O should be improved since the application server need only communicate with a UNIX socket on localhost. Basically, for webservers that are not designed for efficient, non-blocking I/O, we will benefit from having NGINX to handle all I/O operations.

## Versions

* Buildpack Version: 0.6
* NGINX version: 1.11.8
* PCRE version: 8.39
* zlib version: 1.2.11
* OpenSSL version: 1.0.2j

## Supported Heroku stacks

* `cedar-14`
* `heroku-16`

## Requirements

* Your webserver listens to the socket at `/tmp/nginx.socket`.
* You touch `/tmp/app-initialized` when you are ready for traffic.
* You can start your web server with a shell command.

## Features

* Unified NXNG/App Server logs.
* [L2met](https://github.com/ryandotsmith/l2met) friendly NGINX log format.
* [Heroku request ids](https://devcenter.heroku.com/articles/http-request-id) embedded in NGINX logs.
* Crashes dyno if NGINX or App server crashes. Safety first.
* Language/App Server agnostic.
* Customizable NGINX config.
* Application coordinated dyno starts.

### Logging

NGINX will output the following style of logs:

```
measure.nginx.service=0.007 request_id=e2c79e86b3260b9c703756ec93f8a66d
```

You can correlate this id with your Heroku router logs:

```
at=info method=GET path=/ host=salty-earth-7125.herokuapp.com request_id=e2c79e86b3260b9c703756ec93f8a66d fwd="67.180.77.184" dyno=web.1 connect=1ms service=8ms status=200 bytes=21
```

### Language/App Server Agnostic

NGINX-buildpack provides a command named `bin/start-nginx` this command takes another command as an argument. You must pass your app server's startup command to `start-nginx`.

For example, to get NGINX and Unicorn up and running:

```bash
$ cat Procfile
web: bin/start-nginx bundle exec unicorn -c config/unicorn.rb
```

### Setting the Worker Processes

You can configure NGINX's `worker_processes` directive via the
`NGINX_WORKERS` environment variable.

For example, to set your `NGINX_WORKERS` to 8 on a PX dyno:

```bash
$ heroku config:set NGINX_WORKERS=8
```

### Customizable NGINX Config

You can provide your own NGINX config by creating a file named `nginx.conf.erb` in the config directory of your app. Start by copying the buildpack's [default config file](config/nginx.conf.erb).

### Customizable NGINX Compile Options

See [scripts/build_nginx.sh](scripts/build_nginx.sh) for the build steps. Configuring is as easy as changing the "./configure" options.

### Application/Dyno coordination

The buildpack will not start NGINX until a file has been written to `/tmp/app-initialized`. Since NGINX binds to the dyno's $PORT and since the $PORT determines if the app can receive traffic, you can delay NGINX accepting traffic until your application is ready to handle it. The examples below show how/when you should write the file when working with Unicorn.

## Setup

Here are 2 setup examples. One example for a new app, another for an existing app. In both cases, we are working with ruby & unicorn. Keep in mind that this buildpack is not ruby specific.

### Existing App

Add Buildpack:

```bash
$ heroku buildpacks:add https://github.com/washos/nginx-buildpack --index 1 --app app_name
```

Update `Procfile`:

```
web: bin/start-nginx bundle exec unicorn -c config/unicorn.rb
```

```bash
$ git add Procfile
$ git commit -m 'Update procfile for NGINX buildpack'
```

Update `config/unicorn.rb`:

```ruby
require 'fileutils'
listen '/tmp/nginx.socket'
before_fork do |server,worker|
	FileUtils.touch('/tmp/app-initialized')
end
```

```bash
$ git add config/unicorn.rb
$ git commit -m 'Update unicorn config to listen on NGINX socket.'
```

Deploy changes:

```bash
$ git push heroku master
```

### New App

```bash
$ mkdir myapp; cd myapp
$ git init
```

`Gemfile`:

```ruby
source 'https://rubygems.org'
gem 'unicorn'
```

`config.ru`:

```ruby
run Proc.new {[200, {'Content-Type' => 'text/plain'}, ["hello world"]]}
```

`config/unicorn.rb`:

```ruby
require 'fileutils'
preload_app true
timeout 5
worker_processes 4
listen '/tmp/nginx.socket', backlog: 1024

before_fork do |server,worker|
	FileUtils.touch('/tmp/app-initialized')
end
```

Install Gems:

```bash
$ bundle install
```

`Procfile`:

```
web: bin/start-nginx bundle exec unicorn -c config/unicorn.rb
```

Create & push Heroku app:

```bash
$ heroku create --stack cedar-14
$ heroku buildpacks:add https://github.com/washos/nginx-buildpack --index 1 --app app_name
$ heroku buildpacks:add heroku/ruby --index 2 --app app_name
$ git add .
$ git commit -am "First commit"
$ git push heroku master
$ heroku logs -t
```

Visit app

```
$ heroku open
```

## Compilation

### `cedar-14`

```
[local ] $ docker run -i -t heroku/cedar:14 /bin/bash
[local ] $ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
<container_id>      heroku/cedar:14     "/bin/bash"         7 seconds ago       Up 5 seconds                            jolly_bardeen
[local ] $ docker cp scripts/build_nginx.sh <container_id>:/tmp/build_nginx.sh
[docker] # sh /tmp/build_nginx.sh
[local ] $ docker cp <container_id>:/tmp/nginx/sbin/nginx bin/nginx-cedar-14
```

### `heroku-16`

```
[local ] $ docker run -i -t heroku/heroku:16-build /bin/bash
[local ] $ docker ps
CONTAINER ID        IMAGE                    COMMAND             CREATED             STATUS              PORTS               NAMES
<container_id>      heroku/heroku:16-build   "/bin/bash"         4 seconds ago       Up 3 seconds                            pedantic_yalow
[local ] $ docker cp scripts/build_nginx.sh <container_id>:/tmp/build_nginx.sh
[docker] # sh /tmp/build_nginx.sh
[local ] $ docker cp <container_id>:/tmp/nginx/sbin/nginx bin/nginx-heroku-16
```

## License
Copyright (c) 2013 Ryan R. Smith
Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
