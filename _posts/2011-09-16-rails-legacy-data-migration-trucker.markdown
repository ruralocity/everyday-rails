---
layout: post
title: "Migrate data to Rails applications with Trucker"
excerpt: "Get legacy data? Trucker makes moving it from old codebases into new Rails apps with relative ease."
tags: legacy data-migration rails-rescue
---

I spent a lot of time this summer working on moving legacy data from old Rails applications into new apps. In both cases my data model had changed significantly, so it wasn't just a matter of copying data over field for field&mdash;I needed a way to map individual fields and associations; as well as perform some occasional data cleanup to make sure everything would work correctly in the new apps. For this task, I worked with a gem called [Trucker](https://github.com/mokolabs/trucker).

I first learned about Trucker at last year's [Ruby Midwest](http://www.rubymidwest.com/) conference in Kansas City. The main project on GitHub hasn't been touched in almost a year, but some folks have created [newer forks](https://github.com/mokolabs/trucker/network) keeping it up (I started from [this fork](https://github.com/gilesbowkett/trucker), which had better-working generators at the time). All the same, there are a few minor annoyances I had to deal with when using Trucker for myself. In this post, I'll walk through what I had to do to get my application set up for Trucker and my data correctly migrated over.

<div class="alert alert-info">
**Minor disclaimer:** This document shouldn't be treated as a step-by-step tutorial. It may represent more of a hack than some people are comfortable putting to practice themselves, but I can attest that it's a hack that works with some cajoling. Use it as a loose guide to hopefully point out places where you'll have to do a little extra work to get things functioning correctly.
</div>

### What is it?

As mentioned, Trucker is a utility for migrating legacy data into a modern Rails application from an old app. In my case, I was moving over from two Rails apps&mdash;one written in Rails 1.2 and another in version 2.3&mdash;but in reality you can move data from _any_ old application provided Rails and ActiveRecord can talk to its database.

### Who's it for?

Trucker is for anyone developing a brand new Rails application, with a need to copy data from some old legacy app. You'll need access to that old data, of course. You'll also need a handle on how a typical Rails application is structured, and be ready to stretch a little beyond that usual structure. In this case you'll be adding a database that's not part of the _development/test/production_ trifecta, creating models that don't inherit from ActiveRecord, writing a few basic rake tasks, and possibly editing the contents of a gem. This can be pretty time-consuming&mdash;I spent a solid few days on this process in one case&mdash;but it beats the alternatives.

### How does it work?

I'll tell you how Trucker worked for _me_&mdash;as I said, I had to tweak some things to work the way I needed them to, but once everything was right it worked great. Trucker does its thing by adding a `Legacy` base class to your app, on top of which you'll create legacy versions of your application's models. In these legacy models you'll map your new models' fields to their legacy counterparts, and use Ruby to do any cleanup that might be necessary.

Since the gem's README was written in a pre-Rails 3 world, its installation instructions are out of date from the get-go. Just follow the now-standard instructions of adding a dependency to your `Gemfile`, followed by the usual `bundle` command.

According to [this presentation](http://www.slideshare.net/mokolabs/migrating-legacy-data-ruby-midwest), it's preferable to handle everything from your production server, but that wasn't an option for me. So I made a copy of the legacy database and moved it to my development box. Then I configured my Rails application to be aware of it, via my `database.yml` file:

{% highlight yaml %}
  legacy:
    adapter: mysql2
    encoding: utf8
    reconnect: false
    database: store_legacy
    pool: 5
    username: root
    password:
    socket: /tmp/mysql.sock
{% endhighlight %}

Next, there's a generator to create a base class for your legacy models and a starter rake task. As I mentioned above, the main fork for Trucker doesn't appear to be in active development these days&mdash;and the generator doesn't work past Rails 2.3 (at least, it didn't for me last summer). [The fork I used](https://github.com/gilesbowkett/trucker) handled this better, but it still had a hole relative to the rake tasks. I'll get to that in a moment.

The good news is you don't _need_ the generator&mdash;there are only a couple of files to create. First, inside your app's `models` directory, create a new directory called `legacy`. In there, add a file called `legacy_base.rb` with the following (from the Trucker gem):

{% highlight ruby %}
  # app/models/legacy_base.rb
  class LegacyBase < ActiveRecord::Base
    self.abstract_class = true
    establish_connection "legacy"

    def migrate
      @new_record = self.class.to_s.gsub(/Legacy/,'::').constantize.new(map)
      @new_record[:id] = self.id
      begin
        @new_record.save!
      rescue Exception => e
        # this is mostly for ActiveRecord Validation errors - if the validation fails, it
        # typically means you need to adjust the validations or the model you're migrating
        # your legacy data into. this is especially useful information when you're migrating
        # user data established with one auth library to a code base which uses another.
        puts "error saving #{@new_record.class} #{@new_record.id}!"
        puts e.inspect
      end 
    end 

  end
{% endhighlight %}

I had to tell Rails 3.0.x about the <doe>legacy</code> directory, in my app's `application.rb`:
  
{% highlight ruby %}
  config.autoload_paths += %W(#{config.root}/app/models/legacy)
{% endhighlight %}

With the `LegacyBase` class in place you can now start building your legacy models. This is where you'll compare your old data to your new schema and set things up accordingly. If you're migrating from a really old application written in another language or framework, or a pre-RESTful Rails application, this can take some time, but it's a good exercise. I actually realized there was an obtuse field in my legacy application that I hadn't ported over to the new version.

[Refer to the examples](https://github.com/mokolabs/trucker/blob/master/README.rdoc) from Trucker's README to get a sense for how your legacy models will be set up. Below is an annotated example from my work, which migrates a table of billing and shipping addresses:

{% highlight ruby %}
  # app/models/legacy/legacy_address.rb
  
  # If your new application has a model named Address,
  # the legacy class will be LegacyAddress.
  class LegacyAddress < LegacyBase
    # Tell Trucker the name of the legacy table.
    set_table_name "addresses"

    # Map the new model's attributes to the legacy database fields:

    def map 
      {
        :name => set_name,                                # use a custom method to set this value; see below
        :contact => self.name,                            # straight copy from old to new
        :street => self.street,
        :city => self.city,
        :state => LegacyState.find(self.state_id).code,   # grab data from the legacy database using another legacy model and a standard find() method
        :zip => self.zip,
        :user_id => self.user_id,                         # transfer the raw data, not the relationships represented
        :company => self.company
      }
    end

    # A simple Ruby method to prepare legacy data for the new schema. In this example,
    # the name field in my new model is required; the corresponding nickname field in
    # the legacy application was not.
    def set_name
      nickname ? nickname : "Default"
    end
  end
{% endhighlight %}

<div class="alert alert-info">
  If your legacy model is like mine above and either ends in an s or has some other odd inflection, read below for resolving pluralization issues.
</div>

With your legacy model in place, it's time to start moving data. When I was working with Trucker the generator didn't create a starter for this, so I added it myself&mdash;it's pretty straightforward.

{% highlight ruby %}
  # lib/tasks/legacy.rake
  namespace :db do
    namespace :migrate do

      desc 'Migrates addresses'
      task :addresses => :environment do
        Trucker.migrate :addresses
      end
      
      # repeat for each legacy model
    end
  end
{% endhighlight %}

To run a given legacy migration (in my example above, addresses), just run `rake db:migrate:nameoflegacymodel`. For me, each migration required a little bit of babysitting so I didn't create an `all` task that handled every migration&mdash;I took care of them individually.

<div class="alert alert-info">
  **Important!** Trucker will erase all existing data from the database table you're migrating to.
  Keep that in mind; back things up first if this makes you leery.
</div>


#### Pluralization problems?

The model I've been using so far as an example, `Address`, was not a cut-and-dry migration. When you run a Trucker migration, it gets the name of the model to use from the rake task, then does a series of conversions between symbol and string, pluralizing and depluralizing, in the file `lib/trucker.rb` (look inside the gem). I had to make adjustments to this file depending on whether or not my model's inflection pattern was straightforward and whether it ended in an s to begin with&mdash;for example, _address_ might be converted to _addresss_ or _addres_. Someday, if I have time, I'd like to revisit this, come up with a universal fix that works properly with any model name, and share it.

#### Cleanup

To keep things from complaining in production I had to remove my legacy models from `app/models` (I kept them inside my application, but outside of the standard Rails directories). Then I removed the line from my `application.rb` file. If you didn't include the dependency inside your `:development` group you'll also want to comment out or remove the reference from your `Gemfile`. You should now be ready to deploy your code. I followed this up by dumping my data from my development database and importing it into production.

### Conclusion

Trucker isn't perfect, and the above process is among the uglier things I've had to do with Rails&mdash;but it could have been much worse. With the proper adjustments and a little patience Trucker performs as advertised. In the end, I was able to move data from relatively complex structures into a familiar Rails and ActiveRecord-style setup in a fraction of the time it would have taken if I'd had to write something from scratch.