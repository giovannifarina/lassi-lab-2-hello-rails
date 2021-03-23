# Part 4: Deploy to the cloud, including the production database

In the Sinatra Hangperson assignment, you already learned how to
deploy a Sinatra app to Heroku.
Deploying a Rails app is very similar, but a few extra steps are
required since most Rails apps use databases.

Apps on Google Cloud can use either MySQL or PostgreSQL database. In this example, we will use PostgreSQL.  For Ruby/Rails apps
to do so, they must include the `pg` gem.  However, we don't want to
use this gem while developing, since we're using SQLite for that.
Gemfiles let you specify that certain gems should only be used in
certain environments.  Rails apps examine the environment variable
`RAILS_ENV` to determine which environment they're running in, to make
decisions such as which database to use (`config/database.yml`) and
which gems to use.
Google Cloud
sets this variable to `production` at deploy time; running tests sets
it to `test`; while you're running your app interactively, it's set to
`development`. 

To specify production-specific gems, you must make **two** edits to
your Gemfile.  First, add this: 

```
group :production do
  gem 'pg', '~> 0.21' # for gcloud deployment
  gem 'appengine'
end
```

(If there is already a `group :production` in your Gemfile, just add
those lines to it.)  `appengine` is a [gem](https://github.com/GoogleCloudPlatform/appengine-ruby) that has some
additional best-practices for deployment that Google Cloud recommends
(Rails 5 and later apps don't need it).

Second, find the line that specifies the `sqlite3` gem, and tell the
Gemfile that that gem should **not** be used in production, by moving
that line into its own group like so:

```
group :development, :test do
  gem 'sqlite3', '~> 1.3.0'
end
```

This second step is necessary because Heroku is set up in such a way
that the `sqlite3` gem simply won't work, so we have to make sure it
is _only_ loaded in development and test environments but _not_ production.

As always when you modify your Gemfile, re-run `bundle install` and
commit the modified `Gemfile` and `Gemfile.lock`.
Note that when you run Bundler, it will still compute dependencies and
versions for gems in the `production` group, but it won't install
them.  Heroku will use `Gemfile.lock` to install the matching versions
of the gems when you deploy.

**Don't we have to modify `config/database.yml` as well?**  You'd
think so, but the way Heroku works, it actually _ignores_
`database.yml` and forces Rails apps to use Postgres.  So modifying
the `production:` section of `database.yml` won't have any effect on Heroku.

The first time you
use the gcloud command line tool, you will have to authenticate yourself using your gcloud
username and password, by saying `gcloud init`. During this initalization, choose to create a new project. Or you can also run the following:

```
gcloud projects create [project-id]
gcloud config set project [project-id]
```

At this point, having committed all changes locally, you should be ready to push your
repository to gcloud for deployment.

Finally, you need to create a new app "container" on Google Cloud to which
you can deploy.   The easiest way to do this is to use the 
gcloud command line tool from within the app's root directory; type 
```
gcloud app create
```

The created app container will then be
associated with your app. 

OK, the moment of truth has arrived.  `gcloud app deploy` to deploy your code to Google Cloud.  Google Cloud will try to
install all the Gems in your gemfile and fire up your app!  You should
be able to view your app at `https://your-app-name.appspot.com`!
Right?

Almost.  If you try this, you'll see an error. It complains that it does not have `app.yml` file. So let's create this file in the root folder.

```
entrypoint: bundle exec rackup --port $PORT
env: flex
runtime: ruby

automatic_scaling:
  min_num_instances: 1
  max_num_instances: 2

env_variables:
  SECRET_KEY_BASE: [SECRET_KEY]
```

When you deploy a Rails app in the production environment, set the environment variable SECRET_KEY_BASE to a secret key that is used to protect user session data. This environment variable is read in the config/secrets.yml file. Now, we need to do the following:
1. Generate a new secret key
    ```
    bundle exec rake secret
    ```
1. Copy the generated secret key and paste it inside `app.yml` file under `SECRET_KEY_BASE`. 

Before we move one, we need to make sure the billing is linked to the project. Go to [Google Cloud Platform console](https://console.cloud.google.com/appengine/start). On the top, select your project-id which you created then click Billing from the left Menu. Make sure the project is linked to your billing account.

You can now try to deploy again:

```
gcloud app deploy
```

Creating the database locally required 2 steps: first we ran the
initial migration to create the `movies` table (schema), then we
seeded the database with some initial data.  We must do these 2 steps
on Google Cloud as well. To do this, we need to create PostgreSQL instance in Google Cloud.

1. Before you can begin using the Cloud SQL you must enable the Admin API. You can enable the API by using the following command:
    ```
    gcloud services enable sqladmin.googleapis.com
    ```
1. Go to Google Cloud Platform to create a new SQL instances [https://console.cloud.google.com/sql/instances](https://console.cloud.google.com/sql/instances). Choose PostgreSQL database and click ENABLE API to enable Compute Engine API to create an instance. Or you can also create the instance from the terminal by running the following:
    ```
    gcloud sql instances create postgres-dbase --database-version POSTGRES_12 --cpu=1 --memory=3840MiB --region=asia-southeast1
    ```
    This create an instance with the name `postgres-dbase` as a PostgreSQL version 12 database. 
1. Create a database in the instance. You can this database `movies` for example.
1. Set the postgres user password for the instance using the following command. Replace `some-password` with your generated random password. 
    ```
    gcloud sql users set-password postgres --instance=postgres-dbase --password=some-password
    ```
    
When deployed, your application uses the Cloud SQL Proxy that is built into the App Engine environment to communicate with your Cloud SQL instance. However, to test your application using your terminal shell, you must install a copy of the Cloud SQL Proxy in the development environment. To do this, open another Terminal and run the following:
```
cd
wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -O cloud_sql_proxy
chmod +x cloud_sql_proxy
```
The first command changes the directory to the home directory where we will download the Cloud SQL Proxy. The second command simply download and rename it as cloud_sql_proxy. The last command is to change the file to make it executable. We then need to create a directory.

```
sudo mkdir /cloudsql
sudo chmod 0777 /cloudsql
```

Now you can get the connection name by running the following command:
```
gcloud sql instances describe postgres-dbase | grep connectionName 
```
You should replace the name after `describe` to your SQL instance name which you created previously. You should see something like the following:
```
connectionName: rottenpot:asia-southeast1:postgres-dbase
```
Now, you can run Cloud SQL Proxy.
```
./cloud_sql_proxy -dir=/cloudsql -instances=rottenpot:asia-southeast1:postgres-dbase
```
You should see the following output:
```
2021/03/23 08:09:48 Listening on /cloudsql/rottenpot:asia-southeast1:postgres-dbase/.s.PGSQL.5432 for rottenpot:asia-southeast1:postgres-dbase
2021/03/23 08:09:48 Ready for new connections
```
Leave the Terminal open and switch back to the other Terminal. 

Now, we need to modify `config/database.yml`. In particular, you need to edit the `production` section of this file.
```
production:
  adapter: postgresql
  encoding: unicode
  pool: 5
  timeout: 5000
  username: postgres
  password: [your-password]
  database: movies
  host:   /cloudsql/[instance connection name]
```
You need to fill in the password and your instance connection name. You also need to add this connection name to your `app.yml` under the setting `cloud_sql_instances`. Your `app.yml` now should contain the following:

```
entrypoint: bundle exec rackup --port $PORT
env: flex
runtime: ruby

env_variables:
  SECRET_KEY_BASE: [SECRET_KEY]

beta_settings:
  cloud_sql_instances: [YOUR_INSTANCE_CONNECTION_NAME]
```

With this setup, we can do the migration in the Google Cloud App Engine. Run the following

```
RAILS_ENV=production bundle exec rake db:create
RAILS_ENV=production bundle exec rake db:migrate
```

Now go verify that you can visit your app, but no movies are listed...
```
RAILS_ENV=production bundle exec rake db:seed
```

...and now your app should work on Google App Engine just as it does locally,
with the seed data.

Voila -- you have created and deployed your first Rails app!


<details>
<summary>
What are the two steps you must take to have your app use a particular
Ruby gem? 
</summary>
<blockquote>
You must add a line to your `Gemfile` to add a gem and re-run `bundle install`.
</blockquote>
</details>

<details>
<summary>
The _first_ time you deploy a particular app on Google Cloud, what steps do
you need to take (assuming you have adjusted your `Gemfile` correctly
as described above)?
</summary>
<blockquote>
Create the app container on Google Cloud; deploy the app to Google Cloud; run the
initial migration(s) to create the database; and if appropriate, seed
the database with initial data.
</blockquote>
</details>


<details>
<summary>
When you make changes to your app code, including adding new
migrations, what must you do to _update_ the existing Google Cloud app?
(HINT: try making a simple change to the app, like changing something
in a view, and see if you can deduce the sequence of steps.)
</summary>
<blockquote>
Commit changes to Git, then <code>gcloud app deploy</code> to
redeploy.  If you created new migrations, you also need to 
<code>heroku run rake db:migrate</code> to apply them on the Heroku side.
</blockquote>
</details>



<div align="center">
<b><a href="Part3.md">&larr; Previous: Part 3</a></b>
</div>

## References

- [Rails on App Engine](https://cloud.google.com/ruby/rails/appengine)
- [RAils using CloudSQL for PostgreSQL](https://cloud.google.com/ruby/rails/using-cloudsql-postgres)
- [Cloud SQL Proxy](https://cloud.google.com/sql/docs/postgres/sql-proxy)
