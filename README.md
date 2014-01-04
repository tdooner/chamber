# Chamber [![Build Status](https://travis-ci.org/m5rk/chamber.png)](https://travis-ci.org/m5rk/chamber) [![Code Climate](https://codeclimate.com/github/m5rk/chamber.png)](https://codeclimate.com/github/m5rk/chamber)

Chamber lets you source your settings from an arbitrary number of YAML files that
values convention over configuration and has an automatic mechanism for working
with Heroku's (or anyone else's) ENV variables.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'chamber'
```

And then execute:

```sh
$ bundle
```

Or install it yourself as:

```sh
$ gem install chamber
```

## Basic Usage

If you're running a Rails app, by default Chamber will look for all of:

* The file `config/credentials.yml`
* The file `config/settings.yml`
* A set of YAML files in the `config/settings` directory

The YAML data will be loaded and you will have access to the settings
through the `Chamber` class.

**Example:**

Given a `settings.yml` file containing:

```yaml
smtp:
  server: "example.com"
  username: "my_user"
  password: "my_pass"
```

can be accessed as follows:

```ruby
Chamber[:smtp][:server]
# => example.com
```

or

```ruby
Chamber.env.smtp.server
# => example.com
```

and will generate the following environment variables if they don't already
exist:

```sh
SMTP_SERVER
SMTP_USERNAME
SMTP_PASSWORD
```

If the environment variables *do* already exist, then their values will not
be overwritten.

### Existing Environment Variables (aka Heroku)

If deploying to a system which has all of your environment variables already
set, you're not going to use the values stored in the YAML files.  Instead,
you're going to want to pull whatever values the environment variables are
storing.

**Example:**

Given a `settings.yml` file containing:

```yaml
smtp:
  server: "example.com"
  username: "my_user"
  password: "my_pass"
```

If an environment variable is already set like so:

```sh
export SMTP_SERVER="myotherserverisapentium.com"
```

Then when you ask Chamber to give you the SMTP server:

```ruby
Chamber[:smtp][:server]
# => "myotherserverisapentium.com"
```

It will return not what is in the YAML file, but what is in the environment
variable.

## Deploying to Heroku

If you're deploying to Heroku, they won't let you upload custom config files. If
you do not have your config files all stored in your repo (which you shouldn't
if some of the information is sensitive), it becomes more difficult to gain
access to that information on Heroku.

To solve this problem, Heroku allows you to set environment variables in your
application.  Unfortunately this has the nasty side effect of being a pain to
deal with.  For one, you have to deal with environment variables with unweildy
names.  For another, it makes the organization of those variables difficult.

Fortunately, Chamber allows you to organize your environment variables in
separate files and access them in easily using hash or object notation, however
at the same time, it provides a convenient Rake task for pushing all of those
configuration settings up to Heroku as environment variables.

When Chamber accesses those same hash/object notated config values, it will
first look to see if an associated environment variable exists.  If it does, it
will use that before any values inside of the config files.

Simply run:

```sh
rake chamber:push:heroku
```

And all of your settings will be converted to environment variable versions
and set on your Heroku app.

If you have more than one Heroku app, you can pass it to the rake task like so:

```sh
rake chamber:push:heroku --app my_heroku_app_name
```

## Advanced Usage

### Method-Based Environment Variable Access

If you aren't a fan of the hash-based access, you can also access them
using methods:

```ruby
Chamber.env.smtp.server
```

### Predicate Methods

When using object notation, all settings have `?` and `_` predicate methods
defined on them.  They work like so:

#### '?' Predicates Check For Falsity

```ruby
  Chamber.env.my_setting                    # => nil
  Chamber.env.my_setting?                   # => false

  Chamber.env.my_other_setting              # => false
  Chamber.env.my_other_setting?             # => false

  Chamber.env.another_setting               # => 'my value'
  Chamber.env.another_setting?              # => true
```

#### '\_' Predicates Allow for Multi-Level Testing

```ruby
  Chamber.env.empty?                        # => true
  Chamber.env.my_setting_group_.my_setting? # => false
```

#### 'key?' Checks For Existence

The `?` method will return false if a key has been set to false or nil. In order
to check if a key has been set at all, use the `key?('some_key')` method
instead.

Notice the difference:

```ruby
  Chamber.env.my_setting                    # => false
  Chamber.env.my_setting?                   # => false

  Chamber.env.key?('my_setting')            # => true
  Chamber.env.key?('my_non_existent_key')   # => false
```

### In Order to Add Advanced Functionality

In any case that you need to set configuration options or do advanced post
processing on your YAML data, you'll want to create your own object for
accessing it.  Don't worry, Chamber will take you 98% of the way there.

Just include it like so:

```ruby
class Settings < Chamber
end
```

Now, rather than using `Chamber[:application_host]` to access your
environment, you can simply use `Settings[:application_host]`.

### Namespacing

If, when running your app, you would like to have certain files loaded only
under specific circumstances, you can use Chamber's namespaces.

**Example:**

```ruby
Chamber.load( :basepath => '/tmp',
              :namespaces => {
                :environment => -> { ::Rails.env } } )
```

For this class, it will not only try and load the file `config/settings.yml`,
it will _also_ try and load the file `config/settings-<environment>.yml`
where `<environment>` is whatever Rails environment you happen to be running.

#### Inline Namespaces

If having a file per namespace value isn't your thing, you can inline your
namespaces.  Taking the example from above, rather than having `settings.yml`,
`settings-development.yml`, `settings-test.yml`, `settings-staging.yml` and
`settings-production.yml`, you could do something like this:

```yaml
# settings.yml

development:
  smtp:
    username: my_development_username
    password: my_development_password`

test:
  smtp:
    username: my_test_username
    password: my_test_password`

staging:
  smtp:
    username: my_staging_username
    password: my_staging_password`

production:
  smtp:
    username: my_production_username
    password: my_production_password`
```

You can even mix and match.

```yaml
# settings.yml

development:
  smtp:
    username: my_development_username
    password: my_development_password`

test:
  smtp:
    username: my_test_username
    password: my_test_password`

staging:
  smtp:
    username: my_staging_username
    password: my_staging_password`
```

```yaml
# settings-production.yml

smtp:
  username: my_production_username
  password: my_production_password`
````

The above will yield the same results, but allows you to keep the production
values in a separate file which can be secured separately.

#### Multiple Namespaces

Multiple namespaces can be defined by passing multiple items to the loader:

```ruby
Chamber.load( :basepath => '/tmp',
              :namespaces => {
                :environment => -> { ::Rails.env },
                :hostname    => -> { ENV['HOST'] } } )
```

When accessed within the `test` environment on a system named `tumbleweed`, it
will load the following files in the following order:

* `settings.yml`
* `settings-test.yml`
* `settings-tumbleweed.yml`

If a file does not exist, it is skipped.

#### What Happens With Duplicate Entries?

Similarly named settings in later files can override settings defined in earlier
files.

If `settings.yml` contains a value:

```yaml
smtp:
  server: "generalserver.com"
```

And then `settings-test.yml` contains this:

```yaml
smtp:
  server: "testserver.com"
```

The when you access the value with `Chamber[:smtp][:server]` you will receive
`testserver.com`.

## Best Practices

### Why Do We Need Chamber?

> Don't store sensitive information in git.

A better way to say it is that you should store sensitive information separate
from non-sensitive information.  There's nothing inherently wrong with storing
sensitive information in git.  You just wouldn't want to store it in a public
repository.

If it weren't for this concern, managing settings would be trivial, easily
solved use any number of approaches; e.g., [like using YAML and ERB in an
initializer](http://urgetopunt.com/rails/2009/09/12/yaml-config-with-erb.html).

### Organizing Your Settings

We recommend starting with a single `config/settings.yml` file. Once this file
begins to become too unwieldy, you can begin to extract common options (let's
say SMTP settings) into another file (perhaps `config/settings/smtp.yml`).

### Keeping Private Settings Private

Obviously the greater the number of files which need to be kept private the more
difficult it is to manage the settings.  Therefore we suggest beginning with one
private file that stores all of your credentials.

### Ignoring Settings Files

I recommend adding a pattern like this to `.gitignore`:

```
# Ignore the environment-specific files that contain the real credentials:
/config/credentials.yml
/config/credentials-*.yml

# But don't ignore the example file that shows the structure:
!/config/credentials-example.yml
```

### Full Example

Let's walk through how you might use Chamber to configure your SMTP settings:

```yaml
# config/settings.yml

stuff:
  not: "Not Related to SMTP"

# config/settings/smtp.yml

smtp:
  headers:
    X-MYAPP-NAME: My Application Name
    X-MYAPP-STUFF: Other Stuff

# config/settings/smtp-staging.yml

smtp:
  username: my_test_user
  password: my_test_password
```

Now you can access both `username` and `headers` off of `smtp` like so:

```ruby
Chamber[:smtp][:headers]
# => { X-MYAPP-NAME: 'My Application Name', X-MYAPP-STUFF: 'Other Stuff' }

Chamber[:smtp][:password]
# => my_test_password
```

## Ideas

* Add a rake task for validating environments (do all environments have the same
  settings?)

## Alternatives

* [figaro](https://github.com/laserlemon/figaro)
* [idkfa](https://github.com/bendyworks/idkfa)
* [settingslogic](https://github.com/binarylogic/settingslogic)

### Others?

I'd love to hear of other gems and/or approaches to settings!

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
