---
layout: post
title:      "CLI Project - prls"
date:       2020-07-15 22:29:15 +0000
permalink:  cli_project_-_prls
---


For the CLI final project, we were required to make a CLI application that would pull data from an external source, allow the user to go at least one level deep, and provide additional data about the user's selection.

One of the biggest headaches I've had working in theatre is finding new shows to work on, or to find the performing rights for a show. My ruby gem is built around this problem. prls is a look-up service that fetches new and featured plays from the main performing rights organizations, and provides additional information for selected plays.

I began by getting my file structure in order. I used Bundler to automatically set up the structure.

```
bundle gem prls
```

This provided me with a standard directory that I could build from. Next came setting up my bin file. I created a new "prls" file in the bin directory and set it up to use ruby, require my environment file, and call a new CLI session that would be built later on. I wanted to know I had the basic structure working before getting too bogged down in the details.

```
#!/usr/bin/env ruby

require_relative '../lib/prls.rb'

PRLS::CLI.new.call
```

From there, I set up my environment file to require my gem and file dependencies. As I built new classes and required additional gems, I made sure to add them to my environment file in the appropriate chronological order. Once the basic structure was loading correctly and printing a dummy output to the terminal, I started working on my CLI. I wanted to keep it DRY and in line with OOP principles so whenever I started writing too much code, I would go ahead and call methods I hadn't written yet to remind me to shift responsibility to those methods.

```
class PRLS::CLI
     def call
		      header
					menu
		 end
		 
#...more methods
```

I knew I would want to eventually create play objects of various rights management classes that inherit from a generic rights management "super" class. This would help share responsibilities among the various objects and classes to ensure no one class is doing too much work.

I created the following classes:

* CLI - to manage the user interface
* PRO - to provide inherited methods universal to individual pro classes
* DPS - to manage play objects featured by Dramatist's Play Service
* CONCORD - to manage play objects featured by Concord Theatricals
* MTI - to manage play objects featured by Music Theater International
* PLAYSCRIPTS - to manage play objects featured by Playscripts, Inc.
* BPP - to manage play objects featured by Broadway Play Publishing, Inc.
* SCRAPER - to fetch information from passed in websites and return data as arrays and hashes to create objects

I started working with the DPS and SCRAPER classes. The methodologies here would apply to several other classes.

First, I knew I wanted to create objects of plays featured by DPS, I wanted to keep track of them, and I wanted to get additional information about them. I settled on a few universal attrs that would make up the information I would want to know about each play.

```
class PRLS::CLI::DPS

     DPS_PLAYS = []
		 
		 attr_accessor :title, :author, :summary, :blurb, :url
		 
		 def self.all
		      DPS_PLAYS
		 end
		 
		 def self.get_plays
		 end
		 
		 def self.list_plays
		 end
		 
		 def self.get_details
		 end

end
```

The DPS class would work hand in hand with the Scraper class, which would process two types of data for each PRO class.

```
class PRLS::CLI::Scraper

    attr_accessor :dps_plays, :dps_content,
                    :mti_plays, :mti_content,
                    :concord_plays, :concord_content,
                    :playscripts_plays, :playscripts_content,
                    :bpp_plays, :bpp_content
end
```

These two datapoints would be collected at two different points of the user's session. Plays would be collected first, and then the content for a selected play would be collected at a later point.

```
    def dps_index(url)
        dps_index = Nokogiri::HTML(open(url))
        @dps_plays = []

        dps_index.css('table td a').each do |play|
            if play != nil
            @dps_plays << {
                :title => play.text,
                :url => play.attribute('href').value
            }
            end
        end
        @dps_plays
    end

    def dps_info(url)
        dps_info = Nokogiri::HTML(open(url))
        @dps_content = {}

        dps_info.css('#single').each do |content|
            @dps_content = {
                :author => content.css('#authorname').text,
                :summary => content.css('#lexisynopsis').text,
                :blurb => content.css('.lexishorttext').text
            }
        end
        @dps_content
    end
```

Each performing rights organization handled their website for featured plays differently, and so ultimately I had to create 10 different Scraper methods, each handling a different aspect.

Once the DPS and Scraper classes were working appropriately, and I was able to interact with those classes while running the bin file, I then copied (yes, copied - to be made DRY later on) much of the functionality of the DPS class to the other PRO classes. Once each PRO class was working correctly, I then refactored my code to shift the universal methods in the PRO class that would then be shared with each individual pro class. After re-factoring, each pro class looked like this:

```
class PRLS::CLI::DPS < PRLS::CLI::PRO

    DPS_PLAYS = []

    def initialize(attributes)
        super
        DPS_PLAYS << self
    end

    def self.all
        DPS_PLAYS
    end

    def self.get_plays
        if self.all.empty?
            url = "https://www.dramatists.com/dps/nowpublished.aspx"
            self.new_from_scrape(PRLS::CLI::Scraper.new.dps_index(url))
        end
    end

    def self.list_plays
        puts ""
        puts "Here are Dramatist's Play Service, Inc.'s featured plays:"
        super
    end

    def self.get_details(index)
        if self.all[index].need_attr?
            self.all[index].add_attr(PRLS::CLI::Scraper.new.dps_info(self.all[index].url))
        end
    end
end
```

Repetition is minimal, and necessary when it is used. I could further re-factor the class constant method and array, but I personally like the clarity of each array being explicitly named.

As code became more and more DRY, I started getting things ready for launch on rubygems.org. Some of the suggestions out there were far more complex and complicated than necessary. So I settled on using the simple:

```
gem build prls.gemspec

#followed by

gem push prls-0.1.1
```

And now, after all of this, you can install prls on your own ruby environment using

```
gem install prls
```

How simple is that? This has been such an awesome experience, figuring out the process and digital tinkering to make sure things work right. I even took the time to include require another gem someone made for formatting a CLI. Check out word_wrap. It's so simple to use and makes the final product much cleaner.

In my off-time, I'm starting some OOP ruby coding to create a little game packaged in a gem that I can play whenever I need a little mental break from going through the labs. Check out dark_and_deep, coming to a ruby environment near you! Sometime in the dark... and deep... future!
