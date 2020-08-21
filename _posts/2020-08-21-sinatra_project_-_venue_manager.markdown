---
layout: post
title:      "Sinatra Project - Venue Manager"
date:       2020-08-21 13:10:48 -0400
permalink:  sinatra_project_-_venue_manager
---


The Sinatra final project tasked us with creating an MVC CRUD web application. It needed to use ActiveRecord associations, with at least one has_many and one belongs_to association. It also needed user authentication, so as to ensure users can only see their own resources.

I started by thinking through a few project ideas. I decided upon a venue management application that would be usable to me on a daily basis during my day job. I manage two performance venues, seeking out and creating bookings with people from the local community (needless to say, not much of that going on with the COVID-19 outbreak). Nevertheless, this app would need to be able to:

* Create venues
* Create tenants
* Create bookings, associating a tenant and a venue through a booking.

This would fulfill the MVP with the following features:

* Multiple models: User, Venue, Tenant, Booking
* Relationships: User has_many Venues, Tenants, Bookings; Tenant has_many Bookings and belongs_to User; Booking belongs_to User; Venue belongs_to User
* User accounts & user validation
* CRUD resources: Venues, tenants, bookings
* Form validation

Once I settled on this idea, I started getting the file structure together. As a test, I wanted to manually build the file structure, using mkdir and cd to build out the directory.

```
Venue Manager
     - app
     -      - controllers
     -      - models
     -      - views
     - config
     - db
     - public
     - spec
     - config.ru
     - CONTRIBUTING.md
     - Gemfile
     - LICENSE.md
     - Rakefile
     - README.md
     - SPECS.md
```

From there, I built out my Gemfile, adding any gems that I thought would be helpful or used.

```
source 'http://rubygems.org'

gem 'sinatra'
gem 'activerecord', :require => 'active_record'
gem 'sinatra-activerecord', :require => 'sinatra/activerecord'
gem 'sinatra-flash'
gem 'rake'
gem 'require_all'
gem 'thin'
gem 'sqlite3'
gem 'shotgun'
gem 'pry'
gem 'tux'
gem 'rspec'
gem 'capybara'

```

And running ``` bundle install ```.

I had some difficulty getting the database set up. I created my migration files using ``` rake db:create_migration ``` to create my Users, Venues, Bookings, and Tenants tables. Trying to run ``` rake db:migrate ``` failed several times. I kept getting the ActiveRecord adapter error. After far too long, I went to the Sinatra docs and followed them exactly. Needless to say, that fixed the issue and my sqlite db was created.

As I started building the controllers for each model, I would follow the CRUD workflow.

```
    before '/venues*' do
        auth
    end

    get '/venues/new' do
        erb :'venues/new'
    end

    post '/venues/new' do
        new_venue = Venue.create(params)
        current_user(session).venues << new_venue
        redirect "/venues/#{new_venue.id}"
    end

    get '/venues' do
        erb :'venues/all'
    end

    get '/venues/:id' do
        @venue = Venue.find(params[:id])
        erb :'venues/venue'
    end

# and so on...
```

**Note: this file was not this clean while building this controller.

As the MVC files were built out, I fired up shotgun to see how the associations were coming along. Seeing things load up correctly was a major boost to my confidence, and over the past few weeks, I started adding features that I figured would be helpful. Employee management, tasks, inventories, etc. As I added features, I was able to see the usability of the app grow.

This frequent adding of features was not always cherries. I became very close with the lsof command, needing to kill ports frequently having them left open in other terminal instances.

One of the MVP features was user authentication. This was relatively painless to incorporate using BCrypt. I added it to my gemfile, and used the ```has_secure_password``` macro to enable ```.authenticate```. It didn't work right off the bat, causing me to explicitly call the BCrypt authentication methods. My cohort lead helped me identify why the macro wasn't working (password/password_digest syntax). Authentication was good to go.

At this point, I had a usable app on my workstation... at home. While I'm working, I'm using a different Mac, I'm using my iPad and iPhone while on the go throughout the venues. I needed to identify a deployment solution to ensure I could access the app and its database, regardless of where I was. Enter Heroku!

Heroku is a fantastic service that hosts apps on their servers, with Github integration! Every push to the master branch can automatically deploy the latest version of your app. I started the connection, but oh, wait...

Build errors. Heroku doesn't support sqlite3. No worries, however. My cohort lead helped explain how to set up my app to use Postgresql, and the connection with Heroku from that point could not have been easier. The app is live on [https://venuemanger.herokuapp.com](https://venuemanager.herokuapp.com)!

Now came styling. I originally used a basic css file that I incorporated using layout and yield, but it did not offer the responsive I wanted (specifically, a collapsible nav), which would require Javascript. For this reason, I incorporated Bootstrap with an override css file to change a few things up. And now, my app looks nice and clean, regardless of what device you are viewing it on.

Working with Sinatra has been such a fantastic experience. I found myself becoming more and more comfortable with the concepts of OOP while working through ActiveRecord and SQL. I'm excited to start developing more and more complex web apps in the future. 
