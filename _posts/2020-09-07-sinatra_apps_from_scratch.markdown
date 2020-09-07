---
layout: post
title:      "Sinatra Apps from Scratch"
date:       2020-09-07 23:41:35 +0000
permalink:  sinatra_apps_from_scratch
---


So much of the curriculum has been: do the legwork before learning about the automation. I wanted to challenge myself to avoid using Corneal in scaffolding out my Sinatra app. I understand the importance of automation, but I needed to demystify the scaffolding process so I could understand how everything worked together.

## Scaffolding

Starting out in terminal, `cd` your way into the directory where your app will live and use `mkdir` to plant your proverbial flag!

```
cd /Users/jwmcl/Development
mkdir app_name
cd app_name
code .
```
Now what? Well, a CRUD app is going to need a few things. Let's start by building what we need. It's so helpful to understand what shell commands do, so let's use those! We've already used `cd` to change us into the correct directory (as above), so we can go ahead and get started.

```
mkdir app #makes a directory
mkdir app/models
mkdir app/views
mkdir app/controllers
touch app/controllers/application_controller.rb #creates (or updates) a file
```
This modular structure isn't totally necessary, but I like to have things organized neatly, and it'll allow us to easily expand and add functionality in the future.

```
touch Gemfile #specify gem dependencies
touch Rakefile #add rake tasks
touch config.ru #required for shotgun to boot into a local host
```
Don't forget an environment file! We'll make a config directory first.

```
mkdir config #holds any configuration files
touch config/environment.rb #loads up everything in the app environment
```
If we're building an app that'll use a db, we'll definitely want to make a db directory as well. Don't worry about migrate, schema, or the db itself. Those will automatically be created when you create your first migration file and eventually migrate it.

```
mkdir db
```
We also probably want a public folder to hold any of our resources, like stylesheets, images, etc. Let's do that and create an empty css file. I'll name it override.css since I'm using it to override Bootstrap template stylings.

```
mkdir public
mkdir public/stylesheets
touch public/stylesheets/override.css #defines override css styles
```
We definitely want to add a spec directory to hold any of our testing files for good TDD. Let's do that and create a spec_helper and spec file as well.

```
mkdir spec
touch spec/spec_helper.rb #loads testing environment
touch spec/app_name_spec.rb #holds the actual tests themselves
```

Let's add a few files that every good app has.

```
touch README.md #features and install information
touch CONTRIBUTING.md #how to contribute to the app
touch LICENSE.md #how to use the app
touch .gitignore #what to ignore when using git
```

## Version control

Speaking of git, now that we have our file structure scaffolded out, let's get set up with git! Go ahead and make your empty repository on GitHub (or whichever commit service you use).

Back in the local environment...

```
git init #establishes the local repository
git add . #stages all changed files
git commit -m 'first commit' #commits changes to be pushed to remote server
git remote add origin remote-url-here #sets remote branch to be pushed to
git remote -v #verifies the remote branch was set up
git push -u origin master #pushes files to remote branch
```

Now that we're set up for git version control, don't forget to commit every 3-7 minutes depending on how much code you've written!

## Dependencies

So we have our scaffold made. Huzzah! But now what? We need to start actually building code! Let's start with our Gemfile, since we need to make sure we are correctly set up Sinatra and our other dependencies!

```
source 'https://rubygems.org

gem 'sinatra' #our DSL
gem 'activerecord', :require => 'active_record' #for db interaction, NOTE* activerecord is the gem, active_record is the library
gem 'sinatra-activerecord', :require => 'sinatra/activerecord' #enables sinatra activerecord rake tasks
gem 'rake' #for db interaction and other tasks we define
gem 'require_all' #to require all the things
gem 'bcrypt' #for user authentication
gem 'sqlite3' #our local db adapter
gem 'thin' #our local web server, works with shotgun
gem 'shotgun' #boots up our local web server
gem 'pry' #for debugging
gem 'tux' #for debugging our db
gem 'rspec' #for tests
gem 'capybara' #for html tests
gem 'rack-test' #our testing server
gem 'database-cleaner', git: 'https://github.com/bmabey/database_cleaner.git' #cleans db for testing
```
Whew, that's a lot! But they all have their purposes. There are a few other gems you may want to use, depending on your situations. I also ended up using:

```
gem 'sinatra-flash' #for error messages to the user
gem 'pg', '0.20' #postgres db for deployment to Heroku (no sqlite3 support)
```
## Getting the environment set up

Run `bundle install` to lock in your gems with the `Gemfile.lock`. Now we set up our environment and config files. Let's start with environment and go down the list!

```
ENV['SINATRA_ENV'] ||= 'development' #loads our Sinatra environment variable, and sets it to "development" if it does not have a value yet

require 'bundler/setup' #requires Bundler
Bundler.require(:default, ENV['SINATRA_ENV']) #requires all gems in our Gemfile, as well as the Sinatra env variable
set :database, {:adapter => 'sqlite3', :database => "db/#{ENV['SINATRA_ENV']}.sqlite3"} #sets our database to use sqlite3 and to make database in our db folder

require_all 'app' #requires all the things in our app directory
```

Next, we'll set up our `config.ru` file.

```
require './config/environment' #loads up our environment file
use Rack::MethodOverride #lets us utilize RESTful routes like PATCH, PUT, DELETE
run ApplicationController #boots up our application!
```

## Get that database working

Let's get our database working now. Let's start by getting our Rakefile set up so we can use `rake` commands.

```
require_relative './config/environment' #loads the environment file
require 'sinatra/activerecord/rake' #loads the Rake extension within Sinatra

desc 'opens a console' #not necessary, but helpful - defines a new rake task to start Pry
task :console do
    Pry.start
end
```

Run ```rake -T``` and your rake commands should show up in the terminal!

Let's create our first migration file. Let's start with a users table.

```
rake db:create_migration NAME=create_users
```

That'll automatically generate a migrate folder within db, as well as the basic template for your migration file. Let's create our table in that file.

```
class CreateUsers < ActiveRecord::Migration[6.0]
   def change
	    create_table :users do |t|
			   t.string :username
				 t.string :password_digest
		  end
	 end
end
```

Go ahead and try migrating!

```
rake db:migrate
```

If it creates the table, great! If not, double check that your database file is correctly defined within your environment file.

## The home stretch

We're so close! We've scaffolded out our directory, defined our depdencies, built our environment files, and even created our first table! Let's get our application controller up and running so we can fire up the shotgun!

```
require './config/environment' #loading up that environment file

class ApplicationController < Sinatra::Base #inherit from Sinatra, other controllers will inherit from ApplicationController
   
	 configure do
	    set :views, 'app/views' #tells Sinatra where to look for our view files
			set :public_dir, 'public' #defines where our public resources live
	 end

   get '/' do
	    erb :index #tells Sinatra to load the index file within views when the browser GETs the '/' url.
	 end

end
```

And that's just about it! You'll want to create an `index.erb` file within views with some dummy text, but you should be able to fire up `shotgun` and see the fruits of your labor on `localhost:9393`.

I hope this has been helpful in anyway. If something doesn't go quite right for you, send me a holler!
