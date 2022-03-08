# Part 1: Set up the new app
<!---
**NOTE:** These instructions assume you're using Ruby >=2.6.4 and
<3.0.0 (2.6.0-2.6.3 do not work with `byebug`),
and Rails 4.2.x.  **The assignment has not been tested with other
versions, and there are files that will almost certainly cause errors
with Rails >=5.**

You will also need `bundler ~>1.17`.  If `bundle -v` fails, `gem
install bundler --version=1.17.3 --no-document`
to install it.  (Normally, though, installing the `rails` gem will
also install `bundler`.)

You also need to run the following commands on the terminal:

```
sudo apt-get update
sudo apt-get install libpq-dev 
sudo apt-get install python3-distutils
```

Finally, the `gcloud` [command line tool](https://cloud.google.com/sdk/docs/install) should be installed in your development
eenvironment; say `gcloud -v` to verify this fact.  
-->

## Create a new Rails app

<!--- You  may find the Rails [online
documentation](https://api.rubyonrails.org/v4) useful during this assignment.
-->

Now that you have Ruby and Rails installed, create a new, empty
Rails app with the command: 
```sh
rails new rottenpotatoes --skip-test-unit --skip-turbolinks --skip-spring
```

The options tell Rails to omit three aspects of the new app:

* Rather than Ruby's `Test::Unit` framework, in a future assignment we will instead
create
tests using the RSpec framework.

* Turbolinks is a piece of trickery that uses AJAX behind-the-scenes to speed
up page loads in your app.  However, it causes so many problems with JavaScript
event bindings and scoping that we strongly recommend against using it.  A well
tuned Rails app should not benefit much from the type of speedup Turbolinks provides.

* Spring is a gem that keeps your application "preloaded" making it
faster to restart once stopped.  However, this sometimes causes
unexpected effects when you make changes to your app, stop and
restart, and the changes don't seem to take effect.  On modern
hardware, the time saved by Spring is minimal, so we recommend against
using it.

If you're interested, `rails new --help` shows more options available
when creating a new app.


If all goes well, you'll see several messages about files being created,
ending with `run bundle install`, which may take a couple of minutes to complete.  
You can now `cd` to the
newly-created `rottenpotatoes` directory, called the **app root**
directory for your new app.  From now on, unless we say otherwise, all
file names will be relative to the app root.  Before going further,
spend a few minutes examining the contents of the new app directory
`rottenpotatoes` to remind yourself with some of
the directories common to all Rails apps.

What about that message `run bundle install`?
Recall that Ruby libraries are packaged as "gems", and the tool
`bundler` (itself a gem) tracks which versions of which libraries a
particular app depends on.
Open the file called `Gemfile` --it might surprise you that there are 
already gem names in this file even though you haven't written any
app code, but that's because Rails itself is a gem and also depends on
several other gems, so the Rails app creation process creates a 
default `Gemfile` for you.  For example, 
you can see that `sqlite3` is listed, because the default
Rails development environment expects to use the SQLite3 database.

OK, now change into the directory of the app name you just created
(probably `rottenpotatoes`) to continue...


<!---
## Work around the SQLite3 gem bug in v1.4

Rails uses the SQLite3 database as the default for development and testing.  
Unfortunately, versions of the SQLite3 gem starting with 1.4.0 introduced
a bug that make it no longer work properly with Rails.  To work around this,
we must force Rails to use an earlier version, namely any version beginning
with 1.3.  Locate the line in the `Gemfile` that specifies the `sqlite3` gem
and change it to read as follows:

`gem 'sqlite3', '~> 1.3.0'`

Then run `bundle update` and verify that its output contains "Using sqlite3 1.3.x" 
where x is any minor version.
-->

## Check your work

To make sure everything works, run the app locally.  (It doesn't do
anything yet, but we can verify that it is running and reachable!)

```sh
cd rottenpotatoes
bin/rails server
```
Open http://127.0.0.1:3000/ on your browser

When you visit the app's home page, you should see the generic Ruby on Rails landing page, 
which is actually being served by your app.  Later we will define our own routes
so that the "top level" page does not default to this banner.

<!---
## Commit your work
At this stage, everything is working and you should initialize a git repo in your app's folder and commit your app.
Here is an [example `.gitignore` file for Rails on Github](https://github.com/github/gitignore/blob/master/Rails.gitignore).
You should frequently commit your code from now on.
-->

<div align="center">
<b><a href="Part2.md">Next: Part 2 &rarr;</a></b>
</div>
