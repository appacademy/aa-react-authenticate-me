# Authenticate Me, Part 3: Deploying Your Rails + React App To Render.com

You will be using [Render.com] to deploy the live, production version of your
Rails-React app.

## Render.com: An overview

Render.com is what's known as a Platform as a Service (PaaS). It provides you a
free domain where you can host your app (`https://<your-app-name>.onrender.com`)
and servers for running your app and its database.

Render sets up your server's runtime environment for you: installing necessary
dependencies, running the appropriate scripts for compiling your code, setting
environment variables, etc. You just need to tell Render **what** you want it to
do; you don't need to worry about the **how**.

Render also integrates with your repos on GitHub. Once you've set up your
application, you only need to `push` to your GitHub repo and Render will
automatically redeploy your app!

Render's free tier does come with certain limitations. For instance, you can
have only one active free PostgreSQL database at a time, each free PostgreSQL
database expires after 90 days, and your free web services cannot exceed 750
total runtime hours per month. For now, however, you don't need to worry about
these limitations.

## Phase 1: Setting up your Rails + React application

Right now, your React application is on a different localhost port than your
Rails application. However, since your React application consists only of
static files that don't need to bundled continuously with changes in production,
your Rails application can serve the React assets in production too.

Running `npm run build` in the __frontend__ folder typically builds these static
files in a __frontend/dist__ folder. For Rails to serve up these files, however,
you instead want to store them in the __public__ folder of your Rails backend.
To achieve this, specify a new output directory under  `build` key in your
__frontend/vite.config.js__:

```js
// frontend/vite.config.js

export default defineConfig(({ mode }) => ({
  plugins:[
    // plugin config
  ]
  server: {
    // server config
  },
  build: {
    outDir: '../public',
    emptyOutDir: true
  }
}));
```

By specifying the build path in this way, you are instructing the script to
overwrite everything (`emptyOutDir: true`) currently in your root's __/public__
folder with the files of the latest build. (Bye-bye __robots.txt__!) Once you've
made this change, run `npm run build` in your __frontend__ directory and confirm
that it populates the top-level __public__ folder with your build files.

Moving forward, you will need to run `npm run build` any time you make changes
to the frontend. In fact, **you should make it a habit to always run `npm run
build` before you push to GitHub.** This is necessary because Render will not be
building your frontend: all your frontend JS and CSS code will already be
pre-compiled into static files, i.e., the files that appear inside the top-level
__public/assets__ folder. You need to re-build those files whenever you change
their frontend source files.

Folders with build files are typically gitignored because they can be large and
the included files can always be regenerated locally. Now, however, you need the
files in __/public__ pushed to GitHub so you can access them in deployment.
Accordingly, add a `!` before `/public` (or add the whole phrase if `/public`
does not already appear) in your root __.gitignore__ file:

```plaintext
!/public
```

The `!` cancels the following pattern from being gitignored. This ensures that
your __/public__ folder will be available for pushes to GitHub even if other
parent directories have __.gitignore__ files that would exclude it.

Next, you need to tell Rails to serve up __public/index.html__ for any route
that does not begin with `api/`. For this, you will need to add another
controller and adjust your __routes.rb__ file.

Starting with the controller, create a new __static_pages_controller.rb__ file
inside your __app/controllers__ folder. All this controller needs to do is
`render file: Rails.root.join('public', 'index.html')`. In fact, you could have
just created a `frontend_index` action in your `ApplicationController` to handle
this case, except that your `ApplicationController` inherits from
`ActionController::API`, and `ActionController::API` **cannot** serve up html
pages. Making a new `StaticPagesController` that inherits from
`ActionController::Base` enables you to keep all of your other controllers
API-specific (i.e., **fast and lean**).

```rb
# app/controllers/static_pages_controller.rb

class StaticPagesController < ActionController::Base
  def frontend_index
    render file: Rails.root.join('public', 'index.html')
  end
end
```

In __config/routes.rb__, add the following catch-all route **after all of your
other routes**:

```rb
#config/routes.rb

get '*path', to: "static_pages#frontend_index"
```

Again, you can check that this worked. Boot up your Rails server and go to
[`http:localhost:5000`]. If everything is correctly configured, you should now
see your React pages showing up on port 5000!

### Getting your app ready for Render

The last configuration task is to create a script that Render can run to build
your project. Create a __render-build.sh__ file in the __bin__ directory and
copy in the following content (including the comments at the top!):

```sh
#!/usr/bin/env bash

# exit on error
set -o errexit

bundle install
rails db:migrate db:seed # if needed
```

You will tell Render to run this script whenever it needs to build your app. The
commands should all look familiar.

Before committing your changes, use the magnifying glass in VSCode's left-hand
menu (or `cmd+shift+F`) to search your project and make sure that you have
removed all `debugger`s, extraneous `print`/`puts`/`p`s, and unnecessary
`console.log`s.

Also, if you use Faker in your seed file, move the `faker` gem to the top level
of your __Gemfile__ so it will be available when you seed your production
database.

You can explicitly set your production database in __config/database.yml__ as
well. Eliminate the `database`, `username`, and `password` production keys and
instead use the following as the value for `production`:

```yml
production:
  <<: *default
  url: <%= ENV['DATABASE_URL'] %>
```

> **Note:** Spacing and indentation matter in __yaml__ files. Make sure to copy
> the spacing above exactly.

Technically, your app will still work even if you do not explicitly set the
production database here because Rails will always give precedence to a present
`DATABASE_URL` environment variable over anything specified in your
__database.yml__ configuration file. Explicitly setting the production database
in your __database.yml__ file, however, will help avoid potential confusion
later.

Finally, you need to equip your app to run on Render's linux platform.
Check your __Gemfile.lock__ to see if it includes `x86_64-linux` under
`PLATFORMS`. If it does not--and it probably doesn't--then you need to run the
following command in your root directory:

```sh
bundle lock --add-platform x86_64-linux
```

Since this command adds the linux platform to your __Gemfile.lock__, you will
need to run it every time you generate a new __Gemfile.lock__.

Your app should now be ready. **Commit all the changes to your `main` branch!**

## Phase 2: Deploy to Render.com

Now that you've prepared your app, it's time to deploy it to Render.com!

Begin by [creating a Render.com account][render-signup] if you have not already
done so.

Render offers two paths to deployment:

* Add a __render.yaml__ file to your root directory specifying your desired
  configuration. Render will use this file to create a _Blueprint_ for your
  application.
* Use Render's website GUI to configure the various components manually.

Instructions for each option appear below. Choose __ONE__ of them and implement
it to deploy your app.

### Option 1: Add a __render.yaml__ file

__In the root directory,__ create a __render.yaml__ file with the following
content:

```yaml
databases:
  - name: authenticateme
    user: authenticateme
    plan: free
    region: oregon

services:
  - type: web
    name: authenticateme
    plan: free
    region: oregon
    branch: main
    env: ruby
    buildCommand: "./bin/render-build.sh"
    startCommand: "rails s"
    envVars:
      - key: DATABASE_URL
        fromDatabase:
          name: authenticateme
          property: connectionString
      - key: RAILS_ENV
        value: production
      - key: RAILS_MASTER_KEY
        sync: false
```

> **Note:** Spacing and indentation matter in **yaml** files. Make sure to copy
> the spacing above exactly. Also, make sure that you do not use punctuation in
> the `user` field.

This __yaml__ file tells Render that you want to set up a web service to
host/run your app and a PostgreSQL database to store information. A few notes:

* The example above sets the `region` to `oregon`. If you are on the East Coast,
  use `ohio` instead. Whichever region you choose, note that your database and
  web service need to specify the same region! (`oregon` is Render's default.)
* The configuration above sets the `RAILS_MASTER_KEY` to `sync: false`. This
  tells Render that the actual value is not stored in the file, so Render will
  provide a space for you to input the value on the Render page where you
  initially install your Blueprint.

Stage and commit all changes to `main`.

Go to your [Render Dashboard] and select `Blueprints` in the top nav bar. Click
to create a `New Blueprint Instance`. If you have not already connected your
GitHub account, click `Connect account` and follow the steps to connect it. (If
you run into issues connecting your GitHub account, use this article to [reset
your GitHub connection].)

Once you have connected your GitHub account, you should see a list of your
available repos. Select the repo with your Authenticate Me app (i.e., the one
with the __render.yaml__ file).

Enter a `Blueprint Name`--"Authenticate Me" should be fine--and your
RAILS_MASTER_KEY (it's the string in __config/master.key__). Click `Apply`.
Render will then start creating your database and web service.

If you want to watch your app build, go back to the Dashboard and click the line
for your app's `Web Service`. You should see a page logging `First deploy
started for ...`. Click on `deploy` to follow the console output as your build
progresses. If all goes well, you should eventually see `Build successful`
followed by `Deploying...`.

You can now skip to the `Visit your site!` section below.

### Option 2: Use Render's website GUI

**Note:** If you followed Option 1 above, you should skip to the `Visit your
site!` section below.

In this option, you will need to set up your PostgreSQL database and your web
service separately.

#### Set up a PostgreSQL database

Go to your [Render Dashboard] and click the `New +` button on the top right.
In the resulting dropdown menu, select `PostgreSQL`.

Enter `authenticateme` as the `Name` of your database. Select the `Region`
closest to you: `Ohio` if in NY, `Oregon` if in SF. Select the `Free` plan.
Leave other fields as their defaults and click `Create Database` at the bottom
of the page.

Render will take you to your new database's information page. Click the
`Connect` button on the right and copy the address for an `Internal Connection`.
You will need this address when setting up your web service.

That's it! Your database should be ready to go.

#### Set up a web service

Go to your [Render Dashboard] and click the `New +` button on the top right.
In the resulting dropdown menu, select `Web Service`.

If you have not already connected your GitHub account, click `Connect account`
and follow the steps to connect it.

Once you have connected your GitHub account, you should see a list of your
available repos. Select the repo with your app.

Fill in the `Name` field with `authenticateme`. You shouldn't need to specify a
`Root Directory`. Select `Ruby` as the runtime `Environment`. Select the
`Region` closest to you: `Ohio` if in NY, `Oregon` if in SF. (Make sure that the
region matches what you entered for your PostgreSQL database!) Leave the branch
as `main`.

Replace the string in the `Build Command` with `bin/render-build.sh`.  
Replace the string in the `Start Command` with `rails s`.  
Select the `Free` plan.

Click `Environment` and then `Add Environment Variables`. Add a key of
`DATABASE_URL` with a value of the `Internal Connection` string that you copied
when creating your database. Next, add a key of `RAILS_ENV` with a value of
`production`. Finally, add a key of `RAILS_MASTER_KEY` with a value of the text
found in your app's __config/master.key__ file. ***Do not add quotation marks to
any of the values.***

Click `Create Web Service`. This will take you to a page where you can see the
console output as your app builds. If all goes well, you should eventually see
`Build successful` followed by `Deploying...`.

## Visit your site

The web address for your site will have the form
`https://your-app-name.onrender.com`--possibly with some extra letters after
your app's name to avoid conflicts--and appear under your app's name near the
top of its web service page. Once your server is running, click your app's
address and visit it live on the web!

**Note:** It can take a couple of minutes for your site to appear, **even after
Render reports that it is `Live`.** Just be patient and refresh every 30 seconds
or so until it appears.

[Render.com]: https://render.com/
[render-signup]: https://dashboard.render.com/register
[Render Dashboard]: https://dashboard.render.com/
[reset your GitHub connection]: https://community.render.com/t/github-id-belongs-to-an-existing-render-user/2411/7
[`http:localhost:5000`]: http:localhost:5000
