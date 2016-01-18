# Coverband

[![Build Status](https://travis-ci.org/danmayer/coverband.svg?branch=master)](https://travis-ci.org/danmayer/coverband)

A gem to measure production code coverage. Coverband allows easy configuration to collect and report on production code coverage. It can be used as Rack middleware, wrapping a block with sampling, or manually configured to meet any need (like coverage on background jobs).

* Allow sampling to avoid the performance overhead on every request.
* Ignore directories to avoid overhead data collection on vendor, lib, etc.
* Take a baseline to get initial app loading coverage.

At the moment, Coverband relies on Ruby's `set_trace_func` hook. I attempted to use the standard lib's `Coverage` support but it proved buggy when sampling or stopping and starting collection. When [Coverage is patched](https://bugs.ruby-lang.org/issues/9572) in future Ruby versions it would likely be better. Using `set_trace_func` has some limitations where it doesn't collect covered lines, but I have been impressed with the coverage it shows for both Sinatra and Rails applications.

###### Success:
After running in production for 30 minutes, we were able very easily delete 2000 LOC after looking through the data. We expect to be able to clean up much more after it has collected more data.

###### Performance Impact

At the moment the performance impact of standard Ruby runtime coverage can be pretty large. Once getting things working. I highly recommend adding [coverband_ext](https://github.com/danmayer/coverband_ext) to the project which should shave the performance overhead down to something very reasonable. The two ways to deal with performance right now are lowering the sample rate and using the C extension. Often for smaller projects using the C extension makes 100% coverage possible.

## Installation

Add this line to your application's Gemfile:

```bash
gem 'coverband'
```

And then execute:

```bash
$ bundle
```

Or install it yourself as:

```bash
$ gem install coverband
```

## Example Output

Since Coverband is [Simplecov](https://github.com/colszowka/simplecov) output compatible it should work with any of the `SimpleCov::Formatter`'s available. The output below is produced using the default Simplecov HTML formatter.

Index Page
![image](https://raw.github.com/danmayer/coverband/master/docs/coverband_index.png)

Details on a example Sinatra app
![image](https://raw.github.com/danmayer/coverband/master/docs/coverband_details.png)

## Notes

* Coverband has been running in production on Ruby 1.9.3, 2.x, 2.1.x on Sinatra, Rails 2.3.x, Rails 3.0.x, Rails 3.1.x, and Rails 3.2.x
* No 1.8.7 support, Coverband requires Ruby 1.9.3+
* There is a performance impact which is why the gem supports sampling. On low traffic sites I am running a sample rate of 20% and on very high traffic sites I am sampling at 1%, which still gives useful data
    * The impact with the pure Ruby coverband can't be rather significant on sampled requests
    * Most of the overhead is in the Ruby coverage collector, you can now use [coverband_ext](https://github.com/danmayer/coverband_ext) to run a C extension collector which is MUCH faster.
    * Using Redis 2.x gem, while supported, is slow and not recommended. It will have a larger impact on overhead performance. Although the Ruby collection dwarfs the redis time, so it likely doesn't matter much.
    * Make sure to ignore any folders like `vendor` and possibly `lib` as it can help reduce performance overhead. Or ignore specific frequently hit in app files for better perf.

## Configuration

### 1. Create Coverband config file

You need to configure cover band you can either do that passing in all configuration options to `Coverband.configure` in block format, or a simpler style is to call `Coverband.configure` with nothing while will load `config/coverband.rb` expecting it to configure the app correctly. Below is an example config file for a Sinatra app:

```ruby
#config/coverband.rb
require 'json'

baseline = Coverband.parse_baseline

Coverband.configure do |config|
  config.root              = Dir.pwd
  if defined? Redis
    config.redis           = Redis.new(:host => 'redis.host.com', :port => 49182, :db => 1)
  end
  config.coverage_baseline = baseline
  config.root_paths        = ['/app/'] # /app/ is needed for heroku deployments
  # regex paths can help if you are seeing files duplicated for each capistrano deployment release
  #config.root_paths       = ['/server/apps/my_app/releases/\d+/']
  config.ignore            = ['vendor','lib/scrazy_i18n_patch_thats_hit_all_the_time.rb']
  # Since rails and other frameworks lazy load code. I have found it is bad to allow
  # initial requests to record with coverband. This ignores first 15 requests
  config.startup_delay     = Rails.env.production? ? 15 : 2
  config.percentage        = Rails.env.production? ? 30.0 : 100.0

  config.logger            = Rails.logger

  #stats help you collect how often you are sampling requests and other info
  if defined? Statsd
    config.stats           = Statsd.new('statsd.host.com', 8125)
  end
  # config options false, true, or 'debug'. Always use false in production
  # true and debug can give helpful and interesting code usage information
  # they both increase the performance overhead of the gem a little.
  # they can also help with initially debugging the installation.
  config.verbose           = Rails.env.production? ? false : true
end
```

### 2. Configuring Rake

Either add the below to your `Rakefile` or to a file included in your Rakefile such as `lib/tasks/coverband` if you want to break it up that way.

```ruby
require 'coverband'
Coverband.configure
require 'coverband/tasks'
```
This should give you access to a number of cover band tasks

```bash
bundle exec rake -T coverband
rake coverband:baseline      # record coverband coverage baseline
rake coverband:clear         # reset coverband coverage data
rake coverband:coverage      # report runtime coverband code coverage
```

The default Coverband baseline task will try to detect your app as either Rack or Rails environment. It will load the app to take a baseline reading. If the baseline task doesn't load your app well you can override the default baseline to create a better baseline yourself. Below for example is how I take a baseline on a pretty simple Sinatra app.

```ruby
namespace :coverband do
  desc "get coverage baseline"
  task :baseline_app do
    Coverband::Reporter.baseline {
      require 'sinatra'
      require './app.rb'
    }
  end
end
```

To verify that rake is working run `rake coverband:baseline`
then run `rake coverband:coverage` to view what your baseline coverage looks like before any runtime traffic has been recorded.

### 3. Configure Rack to use the Coverband middleware

The middleware is what makes Coverband gather metrics when your app runs.
I setup Coverband in my rackup `config.ru` you can also set it up in rails middleware, but it may miss recording some code coverage early in the rails process. It does improve performance to have it later in the middleware stack. So there is a tradeoff there.

#### For Sinatra apps

For the best coverage you want this loaded as early as possible. I have been putting it directly in my `config.ru` but you could use an initializer, though you may end up missing some boot up coverage.

```ruby
require File.dirname(__FILE__) + '/config/environment'

require 'coverband'
Coverband.configure

use Coverband::Middleware
run ActionController::Dispatcher.new
```

#### For Rails apps

Create an initializer file

```ruby
# config/initializes/coverband_middleware.rb

# Configure the Coverband Middleware
require 'coverband'
Coverband.configure
```

Then add the middleware to your Rails rake middle ware stack:

```ruby
# config/application.rb
[...]

module MyApplication
  class Application < Rails::Application
    [...]

    # Coverband use Middleware
    config.middleware.use Coverband::Middleware

  end
end
```

Note: To debug in development mode, I recommend turning verbose logging on `config.verbose = true` and passing in the Rails.logger `config.logger = Rails.logger` to the Coverband config. This makes it easy to follow in development mode. Be careful to not leave these on in production as they will effect performance.

## Usage

1. Start your server with `rails s` or `rackup config.ru`.
2. Hit your development server exercising the endpoints you want to verify Coverband is recording (you should see debug outputs in your server console)
3. Run `rake coverband:coverage` again, previously it should have only shown the baseline data of your app initializing. After using it in development it should show increased coverage from the actions you have exercised.

Note: if you use `rails s` and data aren't reccorded, make sure it is using your `config.ru`.

## Example apps

- [Rails app](https://github.com/arnlen/rails-coverband-example-app)
- [Sinatra app](https://github.com/danmayer/churn-site)
- [Non rack ruby app](https://github.com/danmayer/coverband_examples)

### Manual configuration (for example for background jobs)

It is easy to use Coverband outside of a Rack environment. Make sure you configure Coverband in whatever environment you are using (such as `config/initializers/*.rb`). Then you can hook into before and after events to add coverage around background jobs, or for any non Rack code.

For example if you had a base Resque class, you could use the `before_perform` and `after_perform` hooks to add Coverband

```ruby
require 'coverband'
Coverband.configure

def before_perform(*args)
  if (rand * 100.0) <= Coverband.configuration.percentage
    @recording_samples = true
    Coverband::Base.instance.start
  else
    @recording_samples = false
  end
end

def after_perform(*args)
  if @recording_samples
    Coverband::Base.instance.stop
    Coverband::Base.instance.save
  end
end
```

In general you can run Coverband anywhere by using the lines below. This can be useful to wrap all cron jobs, background jobs, or other code run outside of web requests. I recommend trying to run both background and cron jobs at 100% coverage as the performance impact is less important and often old code hides around those jobs.


```ruby
require "coverband"
Coverband.configure

coverband = Coverband::Base.instance

#manual
coverband.start
coverband.stop
coverband.save

#sampling
coverband.sample {
  #code to sample coverband
}
```

### Clearing Line Coverage Data

After a deploy where code has changed.
The line numbers previously recorded in Redis may no longer match the current state of the files.
If being slightly out of sync isn't as important as gathering data over a long period,
you can live with minor inconsistency for some files.

As often as you like or as part of a deploy hook you can clear the recorded Coverband data with the following command.

```ruby
# defaults to the currently configured Coverband.configuration.redis
Coverband::Reporter.clear_coverage
# or pass in the current target redis
Coverband::Reporter.clear_coverage(Redis.new(:host => 'target.com', :port => 6789))
```
You can also do this with the included rake tasks.


### Verbose debug mode for development

If you are trying to debug locally wondering what code is being run during a request. The verbose modes `config.verbose = true` and `config.verbose = 'debug'` can be useful. With true set it will output the number of lines executed per file, to the passed in log. The files are sorted from least used file to most active file. I have even run that mode in production without much of a problem. The debug verbose mode outputs both file usage and provides the number of calls per line of code. For example if you see something like below which indicates that the `application_helper` has 43150 lines executed. That might seem odd. Then looking at the breakdown of `application_helper` we can see that line `516` was executed 38,577 times. That seems bad, and is likely worth investigating perhaps memoizing or cacheing is required.

    config.verbose = 'debug'

    coverband file usage:
      [["/Users/danmayer/projects/app_name/lib/facebook.rb", 6],
      ["/Users/danmayer/projects/app_name/app/models/some_modules.rb", 9],
      ...
      ["/Users/danmayer/projects/app_name/app/models/user.rb", 2606],
      ["/Users/danmayer/projects/app_name/app/helpers/application_helper.rb",
      43150]]

    file:
      /Users/danmayer/projects/app_name/app/helpers/application_helper.rb =>
      [[448, 1], [202, 1],
      ...
     [517, 1617], [516, 38577]]

### Merge coverage data over time

If you are clearing data on every deploy. You might want to write the data out to a file first. Then you could merge the data into the final results later.

```ruby
data = JSON.generate Coverband::Reporter.get_current_scov_data
File.write("blah.json", data)
# Then later on, pass it in to the html reporter:
data = JSON.parse(File.read("blah.json"))
Coverband::Reporter.report :additional_scov_data => [data]
```

### Known issues

* `set_trace_func` isn't perfect in sending each line of code executed and can be a bit wonky in a few places. Such as missing the `end` lines in code blocks. If you notice examples of this send them to me.
* If you don't have a baseline recorded your coverage can look odd like you are missing a bunch of data. It would be good if coverband gave a more actionable warning in this situation.
* If you have simplecov filters, you need to clear them prior to generating your coverage report. As the filters will be applied to coverband as well and can often filter out everything we are recording.
* the line numbers reported for `ERB` files are often off and aren't considered useful. I recommend filtering out .erb using the `config.ignore` option.

## TODO

* Fix network performance by logging to files that purge later (like NR) (far more time lost in set_trace_func than sending files, hence not a high priority, but would be cool)
* Add support for [zadd](http://redis.io/topics/data-types-intro) so one could determine single call versus multiple calls on a line, letting us determine the most executed code in production.
* Possibly add ability to record code run for a given route
* Improve client code api, around manual usage of sampling (like event usage)
* Provide a better lighter example app, to show how to use Coverband.
  * blank rails app
  * blank Sinatra app
* report on Coverband files that haven't recorded any coverage (find things like events and crons that aren't recording, or dead files)
* ability to change the coverband config at runtime by changing the config pushed to the Redis hash. In memory cache around the changes to only make that call periodically.

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

## Resources

These notes of kind of for myself, but if anyone is seriously interested in contributing to the project, these resorces might be helpfu. I learned a lot looking at various existing projects and open source code.

##### Ruby Std-lib Coverage

* [Fixed bug causing segfaults on 1.9.X](https://www.ruby-forum.com/topic/1811306)
* [Current Coverage Bug causing issues on 2.1.1](https://bugs.ruby-lang.org/issues/9572)
* [Ruby Coverage docs](http://www.ruby-doc.org/stdlib-1.9.3/libdoc/coverage/rdoc/Coverage.html)

##### Other

* [erb code coverage](http://stackoverflow.com/questions/13030909/how-to-test-code-coverage-for-rails-erb-templates)
* [more erb code coverage](https://github.com/colszowka/simplecov/issues/38)
* [erb syntax](http://stackoverflow.com/questions/7996695/rails-erb-syntax) parse out and mark lines as important
* [ruby 2 tracer](https://github.com/brightbox/deb-ruby2.0/blob/master/lib/tracer.rb)
* [coveralls hosted code coverage tracking](https://coveralls.io/docs/ruby) currently for test coverage but might be a good partner for production coverage
* [simplecov walk through](http://www.tamingthemindmonkey.com/2011/09/27/ruby-code-coverage-using-simplecov) copy some of the syntax sugar setup for cover band
* [Jruby coverage bug](http://jira.codehaus.org/browse/JRUBY-6106?page=com.atlassian.jira.plugin.system.issuetabpanels:changehistory-tabpanel)
* [learn from oboe ruby code](https://github.com/appneta/oboe-ruby#writing-custom-instrumentation)
* [learn from stackprof](https://github.com/tmm1/stackprof#readme)
* I believe there are possible ways to get even better data using the new [Ruby2 TracePoint API](http://www.ruby-doc.org/core/TracePoint.html)

## MIT License
See the file license.txt for copying permission.
