# Part 4: Deploy to the cloud, including the production database

## Preparation

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
Google Cloud sets this variable to `production` at deploy time; running tests sets
it to `test`; while you're running your app interactively, it's set to
`development`. 

To specify production-specific gems, you must make **two** edits to
your Gemfile.  First, add this: 

```
group :production do
  gem 'pg', '~> 0.21' # for gcloud deployment
end
```

(If there is already a `group :production` in your Gemfile, just add
those lines to it.)  

Second, find the line that specifies the `sqlite3` gem, and tell the
Gemfile that that gem should **not** be used in production, by moving
that line into its own group like so:

```
group :development, :test do
  gem 'sqlite3', '~> 1.3.0'
end
```

This second step is necessary because many cloud hosting services are set up in such a way that the `sqlite3` gem simply won't work, so we have to make sure it
is _only_ loaded in development and test environments but _not_ production.

As always when you modify your Gemfile, re-run `bundle install` and
commit the modified `Gemfile` and `Gemfile.lock`.
Note that when you run Bundler, it will still compute dependencies and
versions for gems in the `production` group, but it won't install
them.  Heroku will use `Gemfile.lock` to install the matching versions
of the gems when you deploy.

**Don't we have to modify `config/database.yml` as well?**  You'd
think so, but the way Google Cloud works, it actually _ignores_
`database.yml` and forces Rails apps to use Postgres.  So modifying
the `production:` section of `database.yml` won't have any effect on Google Cloud.

The first time you
use the gcloud command line tool, you will have to authenticate yourself using your gcloud username and password, by saying `gcloud init`. During this initalization, choose to create a new project. Or you can also run the following:

```
gcloud projects create [project-id]
gcloud config set project [project-id]
```

Make sure that billing is enabled for your Cloud project. [Check whether your project is linked to a billing account](https://cloud.google.com/billing/docs/how-to/modify-project). Remember to use the billing account that contains your given credits. 

Enable the Cloud Run, Cloud SQL, Cloud Build, Secret Manager, and Compute Engine APIs. [Enable the API](https://console.cloud.google.com/flows/enableapi?apiid=run.googleapis.com,sql-component.googleapis.com,sqladmin.googleapis.com,compute.googleapis.com,cloudbuild.googleapis.com,secretmanager.googleapis.com).


## Set up a Cloud SQL for PostgreSQL Instance

Follow the steps to create PostgreSQL instance.
1. In the Cloud Console, go to the Cloud SQL Instances page. [CloudSQL Instance page](https://console.cloud.google.com/sql/instances?project=_).
1. Click Create Instance.
1. Click Choose PostgreSQL.
1. In the Instance ID field, enter a name for the instance (`INSTANCE_NAME`).
1. In the Password field, enter a password for the postgres user.
1. Choose Singapore for region and "Single Zone" for Zone availability.
1. Use the default values for the other fields.
1. Click Create Instance.

Note that it may take sometime for the instance create to complete.

## Create a Database
Follow the steps below to create a new database.
1. In the Cloud Console, go to the Cloud SQL Instances page. [Go to the Cloud SQL Instances page](https://console.cloud.google.com/sql/instances?project=_).
1. Select the `INSTANCE_NAME` instance.
1. Go to the Databases tab.
1. Click Create database.
1. In the Database name dialog, enter `DATABASE_NAME`.
1. Click Create.

## Create a User
Generate a random password for the database user, and write it to a file called `dbpassword`:
```
cat /dev/urandom | LC_ALL=C tr -dc '[:alpha:]'| fold -w 50 | head -n1 > dbpassword
```

1. In the Cloud Console, go to the Cloud SQL Instances page. [Go to the Cloud SQL Instances page](https://console.cloud.google.com/sql/instances?project=_).
1. Select the `INSTANCE_NAME` instance.
1. Go to the Users tab.
1. Click Add User Account.
1. Under the Built-in Authentication dialog:
    1. Enter the user name `DATABASE_USERNAME`.
    1. Enter the content of the `dbpassword` file as the password `PASSWORD`.
1. Click Add.

## Set up a Cloud Storage bucket

1. In the Cloud Console, go to the Cloud Storage Browser page. [Go to Browser](https://console.cloud.google.com/storage/browser).
1. Click Create bucket.
1. On the Create a bucket page, enter your bucket information. To go to the next step, click Continue.
    1. For Name your bucket, enter a name that meets the bucket naming requirements.
    1. Select "Region" for Location Type, and choose "asia-southeast1". This will be referred to as `MEDIA_BUCKET` in subsequent steps.
    1. For Choose a default storage class for your data, select the following: Standard.
    1. For Choose how to control access to objects, select Uniform for an Access control option. Make sure you **uncheck** public access prevention.
1. Click Create.

After creating a bucket, to make the uploaded images public, change the permissions of image objects to be readable by everyone.
1. In the Google Cloud Console, go to the Cloud Storage Browser page. [Go to Browser](https://console.cloud.google.com/storage/browser).
1. In the list of buckets, click on the name of the bucket that you want to make public.
1. Select the Permissions tab near the top of the page.
1. Click the Add members button.
1. The Add members dialog box appears.
1. In the New members field, enter `allUsers`.
1. In the Select a role drop down, select the Cloud Storage sub-menu, and click the `Storage Object Viewer` option.
1. Click Save.

## Setting up secret.yml

Open the file `config/secret.yml` and add the following line starting with `gcp`. Copy the password you have generated and previously stored in `dbpassword` into the value after `db_password`.

```
secret_key_base: GENERATED_VALUE
gcp:
  db_password: PASSWORD
```

## Connect Rails app to production database and storage

This tutorial uses a PostgreSQL instance as the production database and Cloud Storage as the storage backend. For Rails to connect to the newly created database and storage bucket, you need to specify all the information needed to access them in the `.env` file. Our `.env` file contains the configuration for the application environment variables. The application will read this file using the dotenv gem. 

1. To configure the Rails app to connect with the database and storage bucket, open the `.env` file.
1. Modify the `.env` file configuration to the following:
  ```
  PRODUCTION_DB_NAME: DATABASE_NAME
  PRODUCTION_DB_USERNAME: DATABASE_USERNAME
  CLOUD_SQL_CONNECTION_NAME: PROJECT_ID:REGION:INSTANCE_NAME
  GOOGLE_PROJECT_ID: PROJECT_ID
  STORAGE_BUCKET_NAME: PROJECT_ID-MEDIA_BUCKET
  ```

## Grant Cloud Build access to Cloud SQL
1. In the Cloud Console, go to the Identity and Access Management page. [Go to the Identity and Access Management page](https://console.cloud.google.com/iam-admin/iam?project=_).
1. To edit the entry for `PROJECTNUM@cloudbuild.gserviceaccount.com` member, click create Edit Member.
1. Click Add another role
1. In the Select a role dialog, select `Cloud SQL Client`.
1. Click Save

## Dockerfile

Create a `Dockerfile` in the root directory containing the following.
```
# Use the official lightweight Ruby image.
# https://hub.docker.com/_/ruby
FROM ruby:2.6.6 AS rails-toolbox

RUN (curl -sS https://deb.nodesource.com/gpgkey/nodesource.gpg.key | gpg --dearmor | apt-key add -) && \
    echo "deb https://deb.nodesource.com/node_14.x buster main"      > /etc/apt/sources.list.d/nodesource.list && \
    apt-get update && apt-get install -y nodejs lsb-release

RUN (curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -) && \
    echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list && \
    apt-get update && apt-get install -y yarn

# Install production dependencies.
WORKDIR /app

COPY Gemfile Gemfile.lock ./

RUN apt-get update && apt-get install -y libpq-dev && apt-get install -y python3-distutils
RUN gem install bundler && \
    bundle config set --local deployment 'true' && \
    bundle config set --local without 'development test' && \
    bundle install

# Copy local code to the container image.
COPY . /app

ENV RAILS_ENV=production
ENV RAILS_SERVE_STATIC_FILES=true
# Redirect Rails log to STDOUT for Cloud Run to capture
ENV RAILS_LOG_TO_STDOUT=true
ENV SECRET_KEY_BASE=<YOUR SECRET_KEY_BASE>

# pre-compile Rails assets with master key
RUN bundle exec rake assets:precompile


ENV RAILS_ENV=production

RUN bundle exec rake db:create
RUN bundle exec rake db:migrate
RUN bundle exec rake db:seed

EXPOSE 8080
CMD ["bin/rails", "server", "-b", "0.0.0.0", "-p", "8080"]
```

You will need to modify the value for the environment variable `SECRET_KEY_BASE`. When you deploy a Rails app in the production environment, set the environment variable SECRET_KEY_BASE to a secret key that is used to protect user session data. This environment variable is read in the `config/secrets.yml` file. Now, we need to do the following:
1. Generate a new secret key. It should output something like a long random string.
    ```
    bundle exec rake secret
    ```
1. Copy the generated secret key and paste it inside `Dockerfile` file under `<YOUR_SECRET_KEY_BASE>`. 

## Creating YAML for Cloud Run

Create a new file called `cloudbuild.yaml` at the root directory. Copy and paste the following.

```
# [START cloudrun_rails_cloudbuild]
steps:
  - id: "build image"
    name: "gcr.io/cloud-builders/docker"
    entrypoint: 'bash'
    args: ["-c", "docker build -t gcr.io/${PROJECT_ID}/${_SERVICE_NAME} . "]

  - id: "push image"
    name: "gcr.io/cloud-builders/docker"
    args: ["push", "gcr.io/${PROJECT_ID}/${_SERVICE_NAME}"]

  - id: "apply migrations"
    name: "gcr.io/google-appengine/exec-wrapper"
    entrypoint: "bash"
    args:
      [
        "-c",
        "/buildstep/execute.sh -i gcr.io/${PROJECT_ID}/${_SERVICE_NAME} -s ${PROJECT_ID}:${_REGION}:${_INSTANCE_NAME} -- bundle exec rake db:migrate"
      ]
  - id: "run deploy"
    name: "gcr.io/google.com/cloudsdktool/cloud-sdk"
    entrypoint: gcloud
    args:
      [
        "run", "deploy",
        "${_SERVICE_NAME}",
        "--platform", "managed",
        "--region", "${_REGION}",
        "--image", "gcr.io/${PROJECT_ID}/${_SERVICE_NAME}"
      ]

substitutions:
  _REGION: [YOUR REGION ]
  _SERVICE_NAME: [YOUR SERVICE]
  _INSTANCE_NAME: [CLOUD SQL INSTANCE NAME]

images:
  - "gcr.io/${PROJECT_ID}/${_SERVICE_NAME}"
# [END cloudrun_rails_cloudbuild]

```

You need to modify the following:
- Replace `[YOUR REGION]` with `asia-southeast1` or your previous chosen region.
- Replace `[YOUR SERVICE]` with the name of you want to set for your Cloud Run service, e.g. `rails-hangperson`
- Replace `[CLOUD SQL INSTANCE NAME]` with your Cloud SQL instance name you previously created.


## Deploying the app to Cloud Run

One last thing to make sure before you deploy to Cloud Run is to enable Cloud Run Admin. To do that, do the following step:
1. In the Cloud Console, go to the Cloud Build Settings page.
1. Open the Settings page
1. Locate the row with the Cloud Run Admin role and set its Status to ENABLED.
1. In the Additional steps may be required pop-up, click GRANT ACCESS TO ALL SERVICE ACCOUNTS.
![](https://www.dropbox.com/s/dbdacdj1px8dh5g/cloud_run_admin_enable.png?raw=1)

At this point, having committed all changes locally, you should be ready to push your repository to gcloud for deployment. Run the following command.

```
gcloud builds submit
```

Voila -- you have created and deployed your first Rails app!

## Applying Database Migration using Cloud SQL

Currently we put our database migrations into the `Dockerfile`. However, if you created new migrations, you also need to <code>rake db:migrate</code> to apply them on the Google Cloud. To do this, we will use the Cloud SQL Proxy to communicate with your Cloud SQL instance. However, to test your application using your terminal shell, you must install a copy of the Cloud SQL Proxy in the development environment. To do this, open another Terminal and run the following:
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
gcloud sql instances describe rotten-sql | grep connectionName 
```
You should replace the name after `describe` to your SQL instance name which you created previously, e.g. `rotten-sql`. You should see something like the following:
```
connectionName: rottenpotatoes-323902:asia-southeast1:rotten-sql
```
Now, you can run Cloud SQL Proxy.
```
./cloud_sql_proxy -dir=/cloudsql -instances=rottenpotatoes-323902:asia-southeast1:rotten-sql
```
You should see the following output:
```
2021/03/23 08:09:48 Listening on /cloudsql/rottenpotatoes-323902:asia-southeast1:rotten-sql/.s.PGSQL.5432 for rottenpotatoes-323902:asia-southeast1:rotten-sql
2021/03/23 08:09:48 Ready for new connections
```
Leave the Terminal open and switch back to the other Terminal. 

With this setup, we can do the migration in the Google Cloud App Engine. Run the following

```
RAILS_ENV=production bundle exec rake db:migrate
```

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
Commit changes to Git, then <code>gcloud builds submit</code> to
redeploy.  
</blockquote>
</details>



<div align="center">
<b><a href="Part3.md">&larr; Previous: Part 3</a></b>
</div>

## References

- [Rails on Cloud Run](https://cloud.google.com/ruby/rails/run)
- [Rails using CloudSQL for PostgreSQL](https://cloud.google.com/ruby/rails/using-cloudsql-postgres)
- [Cloud SQL Proxy](https://cloud.google.com/sql/docs/postgres/sql-proxy)
