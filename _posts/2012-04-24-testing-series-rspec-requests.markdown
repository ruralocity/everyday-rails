---
layout: post
title: "How I learned to test my Rails applications, Part 5: Request specs"
excerpt: "Integration testing with RSpec request specs make sure your application's various parts are working in cohesion. Here's a primer on getting started."
tags: rspec
---

So far we've added a good amount of test coverage to our contacts manager. We got [RSpec installed and configured](http://everydayrails.com/2012/03/12/testing-series-rspec-setup.html), used factories to generate test data, and set up some unit tests on [models](http://everydayrails.com/2012/03/19/testing-series-rspec-models-factory-girl.html) and [controllers](http://everydayrails.com/2012/04/07/testing-series-rspec-controllers.html). Now it's time to put everything together for integration testing&mdash;in other words, making sure those models and controllers all play nicely with other models and controllers in the application. These tests are called _request specs_ in RSpec&mdash;and yes, they can be daunting if you're still getting comfortable with testing.

The good news is you know almost everything you need to know to write a solid request spec&mdash;they follow the same `describe` and `context` structure you've been using in models and controllers, and you can use Factory Girl to generate test data for them.

There's one last component to complete the basic RSpec toolkit: [Capybara](https://github.com/jnicklas/capybara). If necessary, add Capybara (and another gem we'll use in a moment, [DatabaseCleaner](https://github.com/bmabey/database_cleaner)) to your Gemfile's `:test` group like so, running `bundle install` to install the dependencies:

{% highlight ruby %}
# Gemfile

group :test do
  gem 'faker'
  gem 'capybara'
  gem 'database_cleaner' # more on this shortly
end
{% endhighlight %}

Capybara lets you simulate how a user would interact with your application through the web, using a series of easy-to-understand methods like `click_link`, `fill_in`, and `visit`. What these methods let you do, then, is describe a test scenario for your app&mdash;for example:

{% highlight ruby %}
# spec/requests/contacts_spec.rb

require 'spec_helper'

describe "Contacts" do
  describe "Manage contacts" do
    it "Adds a new contact and displays the results" do
      visit contacts_url
      expect{
        click_link 'New Contact'
        fill_in 'Firstname', with: "John"
        fill_in 'Lastname', with: "Smith"
        fill_in 'home', with: "555-1234"
        fill_in 'office', with: "555-3324"
        fill_in 'mobile', with: "555-7888"
        click_button "Create Contact"
      }.to change(Contact,:count).by(1)
      page.should have_content "Contact was successfully created."
      within 'h1' do
        page.should have_content "John Smith"
      end
      page.should have_content "home 555-1234"
      page.should have_content "office 555-3324"
      page.should have_content "mobile 555-7888"
    end
  end
end
{% endhighlight %}

You should recognize some techniques from previous tutorials&mdash;`describe` puts some structure to the spec (we can break things down further with more `context`, too) and the `expect{}` Proc we checked out in the [controller specs tutorial](http://everydayrails.com/2012/04/07/testing-series-rspec-controllers.html) plays the same role here. Following the Proc, we run a series of tests to make sure the resulting view is displayed in a way we'd expect, using Capybara's not-quite-plain-English-but-still-easy-to-follow style. Check out, too, the `within` block used to specify _where_ to look on a page for specific content&mdash; in this case, within the `<h1>` tag in the `show` view for contacts. You can get pretty fancy with this if you'd like&mdash;more on that in just a moment.

One other thing to point out: Within request specs, I think it's perfectly reasonable to have multiple expectations in a given example. Request specs typically have much greater overhead than your models and controllers, and as such can take a lot longer to run.

Of course, we need to load Capybara into RSpec to make it work, so open your `spec/spec_helper.rb` file and add the following, below any other `require` statements you've got there (and while you're at it, go ahead and require DatabaseCleaner, assuming you added it to your Gemfile and installed the gem already&mdash;if not, do that first):

{% highlight ruby %}
  # spec/spec_helper.rb

  ENV["RAILS_ENV"] ||= 'test'
  require File.expand_path("../../config/environment", __FILE__)
  require 'rspec/rails'
  require 'rspec/autorun'
  require "capybara/rspec"
  require 'database_cleaner'
{% endhighlight %}

So we've verified (with a passing spec) that our user interface for adding contacts is working as planned. Let's get slightly more complicated&mdash;this time we'll add a contact, then use the web interface (via Capybara) to delete it. The spec should look something like this:

{% highlight ruby %}
require 'spec_helper'

describe "Contacts" do
  describe "Manage contacts" do
    # earlier request spec hidden
    
    it "Deletes a contact" do
      contact = Factory(:contact, firstname: "Aaron", lastname: "Sumner")
      visit contacts_path
      expect{
        within "#contact_#{contact.id}" do
          click_link 'Destroy'
        end
      }.to change(Contact,:count).by(-1)
      page.should have_content "Listing contacts"
      page.should_not have_content "Aaron Sumner"
    end
  end
end
{% endhighlight %}

Nothing too complex there&mdash;but what if we'd like to verify we've got the Javascript confirmation dialog wired up correctly? Luckily, Capybara bundles support for the Selenium web driver out of the box. With Selenium, you can simulate web interactions, including Javascript, through a copy of Firefox loaded on your computer. Here's what the modified spec looks like:

{% highlight ruby %}
it "Deletes a contact", js: true do
  DatabaseCleaner.clean
  contact = Factory(:contact, firstname: "Aaron", lastname: "Sumner")
  visit contacts_path
  expect{
    within "#contact_#{contact.id}" do
      click_link 'Destroy'
    end
    alert = page.driver.browser.switch_to.alert
    alert.accept
  }.to change(Contact,:count).by(-1)
  page.should have_content "Listing contacts"
  page.should_not have_content "Aaron Sumner"
end
{% endhighlight %}

Notice what's different: We've added `js: true` to the example, to tell Capybara to use a Javascript-capable driver (Selenium, by default). We've also added a couple of lines within `expect{}` to tell Capybara what to do with the dialog when it pops up.

One more new item: The call to `DatabaseCleaner.clean` in the first line of the example. Typically, RSpec is good about cleaning out your test database for each example, but when you use Selenium you'll need to help it out a bit. That's where DatabaseCleaner comes into play. However, you'll need to tell RSpec to use DatabaseCleaner's truncation "strategy" instead of RSpec's default technique for resetting test data.

Open `spec/spec_helper.rb` again locate the line:

{% highlight ruby %}
config.use_transactional_fixtures = true
{% endhighlight %}

*Comment that line out.* Below it, paste the following replacement strategy:

{% highlight ruby %}
DatabaseCleaner.strategy = :truncation
{% endhighlight %}

With those changes, the request spec will run through Firefox (amaze your friends as the browser launches and works through the steps of your web interface), and you're one step closer to a well-tested application.

## Parting advice

* **Keep it simple:** If you don't get request specs right away, don't worry about it. They require some additional setup and thinking to not just work, but actually test what you need to test. Don't stop testing your models and controllers, though&mdash;building skills at that level will help you grasp more complicated specs sooner rather than later.
* **Make time for testing:** Yes, tests are extra code for you to maintain; that extra care takes time. Plan accordingly, as a feature you may have been able to complete in an hour or two before might take a whole day now. This especially applies when you're getting started with testing. However, in the long run you'll recover that time by working from a more trustworthy code base.
* **Don't revert to old habits!** It's easy to get stuck on a failing test that shouldn't be failing. If you can't get a test to pass, make a note to come back to it&mdash;_and then come back to it_. The RSpec mailing list is helpful, as are books like _[The RSpec Book](http://www.amazon.com/gp/product/1934356379/ref=as_li_ss_tl?ie=UTF8&tag=everrail-20&linkCode=as2&camp=1789&creative=390957&creativeASIN=1934356379)_ and _[Rails Test Prescriptions](http://www.amazon.com/gp/product/1934356646/ref=as_li_ss_tl?ie=UTF8&tag=everrail-20&linkCode=as2&camp=1789&creative=390957&creativeASIN=1934356646)_.

## Conclusion

You've now got all the tools you need to do basic automated testing in your Rails projects, using RSpec, Factory Girl, Capybara, and DatabaseCleaner to help. These are the core tools I use daily as a Rails developer, and the tutorials I've presented here show how _I_ learned to effectively use them to increase my trust in my code. I hope I've been able to help you get started with these tools as well.

There's a lot we haven't covered, though:

* Getting fancy with factories
* Testing cases requiring user login (authentication) or role-checking (authorization)
* Speeding up tests by hitting the database less (mocks and stubs)
* What it means to be a _test-driven developer_
* Other things we could be testing (email, routes, etc.)

I'm happy to announce that **I'm working on a book** to cover these topics! There will be more details in the coming days, but I'm self-publishing to keep it affordable (under 10 American bucks) and intend to provide a complete source example and updates through Rails 4.0. [Get early access now!](http://leanpub.com/everydayrailsrspec)