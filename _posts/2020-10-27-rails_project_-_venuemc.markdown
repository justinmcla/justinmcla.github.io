---
layout: post
title:      "Rails Project - VenueMC"
date:       2020-10-27 13:39:55 -0400
permalink:  rails_project_-_venuemc
---


The Rails project was an absolute blast to work on, but it certainly had its share of headaches along the way. I decided to rebuild my Sinatra project in Rails while extending functionality. A few improvements I wanted to make included file uploads and emails. Rails 6 comes with support for a vast majority of features right out of the box, so let's take a closer look at how these features were implemented.

## File Uploads
ActiveStorage makes file storage easy so users can upload and view their files without much fuss. First, you install ActiveStorage with `rails active_storage:install`. This will create a migrtation file for your DB to add two tables, seen below:

```
class CreateActiveStorageTables < ActiveRecord::Migration[5.2]
  def change
    create_table :active_storage_blobs do |t|
      t.string   :key,        null: false
      t.string   :filename,   null: false
      t.string   :content_type
      t.text     :metadata
      t.bigint   :byte_size,  null: false
      t.string   :checksum,   null: false
      t.datetime :created_at, null: false

      t.index [ :key ], unique: true
    end

    create_table :active_storage_attachments do |t|
      t.string     :name,     null: false
      t.references :record,   null: false, polymorphic: true, index: false
      t.references :blob,     null: false

      t.datetime :created_at, null: false

      t.index [ :record_type, :record_id, :name, :blob_id ], name: "index_active_storage_attachments_uniqueness", unique: true
      t.foreign_key :active_storage_blobs, column: :blob_id
    end
  end
end
```

The first table handles all of the actual files themselves, keeping track of various file data. The second table is polymorphic and attaches the blobs to various records throughout your application. 

Run `rails db:migrate` and they're good to go. You can now use the following macros to add file uploads to a model in your app:

```
class SomeClass < ApplicationRecord
  has_one_attached :avatar
	has_many_attached :images
end
```

You will also need to add these to your strong params so that your forms can accept these fields. Further configuration options include being able to set up where files will be stored. In your `config/storage.yml` file, you can provide authentication information for cloud storage. I'm using Amazon S3, and so my config file looks something like this:

```
amazon:
  service: S3
  access_key_id: <%= ENV['AWS_ACCESS_KEY_ID'] %>
  secret_access_key: <%= ENV['AWS_SECRET_ACCESS_KEY'] %>
  region: <%= ENV['AWS_REGION'] %>
  bucket: <%= ENV['AWS_BUCKET'] %>
```

From here, you'll need to tell the appropriate environment file which storage to use:

```
# config/environments/development.rb
...
config.active_storage.service = :amazon
```

## Emails
ActionMailer allows you to send emails using a variety of methods that you can configure in your environment files. I set up an email address and server using AWS WorkMail and SES to handle emails. To get started, you'll need to set up mail settings for each environment.

```
  # config/environments/development.rb
  # Development Action Mailer settings
  config.action_mailer.raise_delivery_errors = true
  config.action_mailer.asset_host = { host: 'localhost', port: 3000 }
  config.action_mailer.perform_caching = false
  config.action_mailer.perform_deliveries = true
  config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }
  config.action_mailer.delivery_method = :smtp

  config.action_mailer.smtp_settings = {
    address: ENV['AWS_SMTP_SERVER'],
    port: 587,
    user_name: ENV['AWS_SMTP_USERNAME'],
    password: ENV['AWS_SMTP_PASSWORD'],
    authentication: :login,
    enable_starttls_auto: true
  }
```
	
	You'll want to set your asset host and default url options to match whatever server is hosting your app. Then you can generate a mailer controller with any email templates you know you'll want.
	
```
	rails g mailer User welcome_letter
```
	
	This will automatically set up your mailer controllers to correctly inherit, as well as create the welcome_letter option in the UserMailer. It will also automatically create `html` and `txt` versions for each action you specified. This is great because the mailer will bundle both so that the correct version is sent to the receiver depending on the receiver's preferences.
	
I call my mailer after a user successfully signs up.
	
```
 UserMailer.with(user: @user).welcome_letter.deliver_now
```
 
 `.with` allows us to send data to the welcome_letter action to be referenced in the letter itself. `.deliver_now` tells Rails to deliver the email immediately, but you can also set up with `deliver_later` to hand it off to a background job to not hold up the user.
 
