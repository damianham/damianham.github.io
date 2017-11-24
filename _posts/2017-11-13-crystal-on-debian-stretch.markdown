---
layout: post
title:  "Crystal on Debian Stretch"
date:   2017-11-13 19:33:42 -+0700
comments: true
disqus_url: "http://damianham.github.io/2017/11/13/crystal-on-debian-stretch.html"
disqus_identifier: "/2017/11/13/crystal-on-debian-stretch.html"
---
I heard about the [crystal programming language](http://crystal-lang.org/) some time ago but up until now I haven't
had an opportunity to try it out.  For the past few years I have been working mostly with Node.JS and Angular and React but I still do Ruby contracts from time to time.  I have been building apps with Ruby on Rails since 2008 and it is a great framework that can get your web application up and running quickly.  Convention over configuration means you don't have to configure every little detail before your site will work.  Crystal gives us that lovely Ruby syntax combined with the speed of a statically typed compiled language, awesome!

Check out these [benchmarks](https://github.com/kostya/benchmarks) to see how impressive the performance is but here is a glimpse to wet your appetite.

![benchmarks](http://i.imgur.com/cjoXet6.png)

One of the issues with Ruby and Ruby on Rails apps is that they can't be distributed to customers without leaving your intellectual property wide open to copying.  Some time ago I built a 3 stage process to bootstrap a Rails application compiled with JRuby and running in the JVM.  It worked but it was complicated and of course requires a compatible JVM on the client workstation.  Crystal gives us an opportunity to distribute self contained Ruby applications to customers in executable binary format.

Crystal installs and runs without issues on Mac OSX however I had a few problems installing on Debian Stretch, but no worries - they are easily overcome.  There isn't (at this time) an installable debian package for Stretch so I found the best option is to install with linuxbrew.  Read on for details of how I got it working on Debian Stretch.

First of all install linuxbrew with
```sh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/Linuxbrew/install/master/install.sh)"
```
This downloads a ruby install script and runs it with your ruby interpreter.  Quick Tip - if you don't already have ruby installed then install via [rbenv](https://github.com/rbenv/rbenv) which is great for managing multiple versions of ruby and gems.  The Ruby script for Linux users will install linuxbrew to either /home/linuxbrew/.linuxbrew or $HOME/.linuxbrew.  If you have recently ran sudo in the terminal then it will install to /home/linuxbrew/.linuxbrew otherwise you will be prompted for your password to run sudo (and install to /home/linuxbrew/.linuxbrew) or Press Control-D to install to $HOME/.linuxbrew.

We are promted to update our environment startup profile so that man and info will find the data for linuxbrew installed programs and ensure the linuxbrew bin folder is in our path.  I use bash so I edited $HOME/.bash_profile and added
```
export PATH=/home/linuxbrew/.linuxbrew/bin:$PATH
```

to put the linuxbrew binaries at the front of my path so they are found first.  Source .bash_profile or type this line into the current terminal to update the path now.  Linuxbrew suggests we install gcc and who are we to argue so the next command is
```sh
brew install gcc
```

This installs gcc and stdlibs into the linuxbrew installation folder.  Now we are ready to install crystal and build any dependencies with linuxbrew supplied gcc and libraries, type;
```sh
LD_LIBRARY_PATH=$(brew --prefix)/lib  brew install crystal
```

Once the install has finished you should be able to execute crystal with
```sh
LD_LIBRARY_PATH=$(brew --prefix)/lib crystal --version
```

The next step is to see if we can compile and run an application with crystal.  I am building a REST api with crystal so a simple hello world web app seems like a good starting point.  First initialise a new crystal application and cd into the created folder.
```sh
crystal init app mywebapp
cd mywebapp
```

There is already a good collection of frameworks and libraries that you can use, see [Awesome Crystal](https://github.com/veelenga/awesome-crystal). If you are looking for a web application framework then there are a number to [choose from](https://github.com/veelenga/awesome-crystal#web-frameworks).  For the web app I am currently building I went with [Kemal framework](http://kemalcr.com/) which will make anyone familiar with Sinatra feel at home.  We need to add kemal as a project dependency, open shard.yml and add these lines
```yaml
dependencies:
  spec-kemal:
    github: kemalcr/spec-kemal
  kemal:
    github: kemalcr/kemal
```

Then in the terminal install the shard dependencies with
```sh
shards install
```
[Shards](https://github.com/crystal-lang/shards) is a dependency manager similar to ruby gems or npm and it came along with the crystal install (crystal deps does the same thing).  Let's add a little hello world web page.  Open src/mywebapp.cr in an editor and repalce the content with

```ruby
require "kemal"

module Mywebapp
  # Matches GET "http://host:port/"
  get "/" do
    "Hello World!"
  end

  # Creates a WebSocket handler.
  # Matches "ws://host:port/socket"
  ws "/socket" do |socket|
    socket.send "Hello from Kemal!"
  end
end

Kemal.run
```
Now in the terminal type
```sh
LD_LIBRARY_PATH=$(brew --prefix)/lib crystal run src/mywebapp.cr
```

If everything is OK you should see the following in the terminal
```
[development] Kemal is ready to lead at http://0.0.0.0:3000
```
Open a browser and visit http://localhost:3000 and say hello to your running crystal application.

A good choice for building a single page application would be [PREACT](https://preactjs.com/) and using Kemal as the JSON api.  For database access check out the list in [awesome-crystal](https://github.com/veelenga/awesome-crystal) for your database driver and ORM needs.  

You have noticed of course that I am prefixing the crystal command with the LD_LIBRARY_PATH=...
You don't want to have to type this all the time do you ?  You could add this to your environment profile however I like to keep things isolated from my base workstation.  I use rbenv for isolated Ruby and Gems installations which neatly sets up my environment on a per project basis but I didn't find anything similar for crystal.

No worries, npm and gulp to the rescue.  I am building the frontend of my apps in Javascript (Preact) bundled with Webpack so I am using npm to manage dependencies.  I can also use npm to run commands. Let's initialise our project with npm and set it up
```sh
npm init
```

creates a package.json.  Open package.json in an editor and set the value of the scripts element;
```json
"scripts": {
  "runapp": "LD_LIBRARY_PATH=/home/linuxbrew/.linuxbrew/lib crystal run src/mywebapp.cr",
  "test": "LD_LIBRARY_PATH=/home/linuxbrew/.linuxbrew/lib KEMAL_ENV=test crystal spec",
  "build": "LD_LIBRARY_PATH=/home/linuxbrew/.linuxbrew/lib crystal build src/mywebapp.cr",
  "release": "LD_LIBRARY_PATH=/home/linuxbrew/.linuxbrew/lib crystal build --release src/mywebapp.cr"
},
```

Now I simply type npm run or npm test to run crystal with the correct libraries.  Finally I setup gulp to test the project each time I modify a source file.
```sh
npm install -g gulp
npm install --save-dev gulp gulp-watch
```

And create a gulpfile.js with this content
```js
const gulp = require('gulp');
const exec = require('child_process').exec;

gulp.task('build', function (done) {
  exec('npm run build', function (err, stdout, stderr) {
    console.log(stdout);
    console.log(stderr);
  });
  done();
});

gulp.task('test', function (done) {
  exec('npm test', function (err, stdout, stderr) {
    console.log(stdout);
    console.log(stderr);
  });
  done();
});

gulp.task('watch', ['test', 'build'], function (done) {

  reporter = 'dot';
  gulp.watch(['./src/**/*'], ['test', 'build']);

  gulp.watch(['./spec/**/*'], ['test']);

  done();
});

gulp.task('default', ['watch']);
```

The next thing to add is running the app from gulp and adding live reload to get a really nice dev environment with a blazingly fast web application, awesome!
