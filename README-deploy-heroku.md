# Authenticate Me, Part 3 (Optional): Deploying Your Rails + React App To Heroku

Use this guide if you want to deploy the live, production version of your
Rails-React app to [Heroku].

## Heroku: An Overview

Heroku is what's known as a Platform as a Service (PaaS). It provides you a free
domain where you can host your app (`<your-app-name>.herokuapp.com`) and
servers for running your app and its database.

What distinguishes Heroku from more bare-bones Infrastructure as a Service
(IaaS) platforms is that it provides numerous tools for easily managing and
scaling your app, doing a lot of the dirty work of network administration for
you.

For instance, Heroku takes care of setting up your server's runtime environment
for you: installing necessary dependencies, running the appropriate scripts for
compiling your code, setting environment variables, etc. You just need to tell
Heroku _what_ you want it to do; you don't need to worry about the _how_.

Another great thing about Heroku is that it uses `git` to manage your source
code. Just like GitHub can host remote repos, you'll create a remote repo on
Heroku for your project. All you will need to do then is `push` to this repo!

## Phase 1: Setting up your Rails + React application

Right now, your React application is on a different localhost port than your
Rails application. However, since your React application consists only of
static files that don't need to bundled continuously with changes in production,
your Rails application can serve the React assets in production too.

Running `npm run build` in the __frontend__ folder typically builds these static
files in a __frontend/dist__ folder. For Rails to serve up these files, however,
you instead want to store them in the __public__ folder of your Rails backend.
To achieve this, specify a new output directory in your
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
}));
```

By specifying the build path in this way, you are instructing the script to
overwrite everything currently in the top-level __public__ folder with the files
of the latest build. (Bye-bye __robots.txt__!) Once you've made this change, run
`npm run build` in your __frontend__ directory, and confirm that it populates
the top-level __public__ folder with your build files.

Moving forward, you will need to run `npm run build` any time you make changes
to the frontend. In fact, **you should make it a habit to always run `npm run
build` before you push to GitHub.** This is necessary because Heroku will not be
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

Next, you need to tell Rails to serve up __public/index.html__ for any route that
does not begin with `api/`. For this, you will need to add another controller
and adjust your __routes.rb__ file.

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

### Getting your app ready for Heroku

When you push a repo to Heroku, Heroku examines the root directory to see what
kind of app it is. In this case, the __Gemfile__ will signal to Heroku that your
project is a Ruby app. The `railties` gem inside the __Gemfile__ lets Heroku
know that our project is also a Rails 7 app. Heroku will accordingly proceed to
run the usual build steps for deploying a Ruby-on-Rails app, from running
`bundle install` to deploying the web server with `rails server`.

The last configuration task is to create a __Procfile__ (no extension).
[__Procfile__s][procfiles] list the commands to run for various process types.
Each line has the form `<process type>: <command>`. For example, you can specify
your app's web server like this: `web: rails server -p $PORT -e $RAILS_ENV`.
Heroku would likely guess that command for a Rails app, but it's always best to
specify explicitly, especially since you have a hybrid app.

More importantly, a __Procfile__ gives you the opportunity to identify commands
to run before deploying a new release. In this case, it will prove convenient to
have Heroku automatically migrate any new migrations before deploying your app.
You can achieve this with a __Procfile__ that looks like this:

```plaintext
web: rails server -p $PORT -e $RAILS_ENV
console: rails console
release: rails db:migrate
```

(The first two lines copy Heroku defaults for Rails apps.)

Before committing your changes, use the magnifying glass in VSCode's left-hand
menu (or `cmd+shift+F`) to search your project and make sure that you have
removed all `debugger`s, extraneous `print`/`puts`/`p`s, and unnecessary
`console.log`s.

Also, if you use Faker in your seed file, move the `faker` gem to the top level
of your __Gemfile__ so it will be available when you go to seed your production
database. Similarly, if you want to use `pry` instead of `irb` for your
production console, move `pry-rails` to the top level of your __Gemfile__. If
you make any changes, `bundle install` to make sure everything still works.

Your app should now be ready. **Commit the changes to your `main` branch!**

## Phase 2: Deploy to Heroku

Now that you've prepared your app, it's time to set up Heroku and push your app
to production!

### Step 1: Create a new Heroku app

+ [Create an account][heroku-signup] on Heroku.

+ Create a new app. From the [dashboard], click `New` > `Create a new app`. Give
  it a unique name, and don't change the region. Click `Create app`.

### Step 2: Add buildpacks

Once it has discerned the type of app you have pushed, Heroku uses _buildpacks_
to know how to proceed. As noted above, it can often determine what buildpack to
use automatically. The Gemfile in your root directory should be enough for
Heroku to discern that your app requires a Ruby buildpack, but if you ever need
to specify a buildpack explicitly--or add multiple buildpacks--you would do it
like this.

To install the Ruby buildpack:

+ Go to the _Settings_ tab in your newly created app's dashboard.
+ Scroll down to _Buildpacks_, click `Add buildpack`, and select
  `heroku/ruby` if it is not already there.

> **Note:** Buildpacks will run in the order you install them.

You can read more about the Ruby buildpack [here][ruby-buildpack].

### Step 3: Add config vars (environment variables)

+ Still in the _Settings_ tab, look for the _Config Vars_ section, and click
  `Reveal Config Vars`. These [configuration variables][config-vars] will be
  included in your app's environment on Heroku.

+ Create a key of `RAILS_MASTER_KEY`. Find the file __config/master.key__ in
  your Rails app, copy the string in that file, and paste it as the value for
  this variable on Heroku. (Heroku won't receive this file when you push, since
  it is git ignored). By giving Heroku access to your master key, your
  production app will have access to any [encrypted
  credentials][rails-credentials] you might create.

### Step 4: Pushing to Heroku

+ Install the Heroku Command Line Interface (CLI) by following [these
  instructions][heroku-cli].

+ Head to the _Deploy_ tab of your Heroku dashboard. Under _Deploy using Heroku
  Git_, find the _Existing Git repository_ header. Copy the command there (it
  should start with `heroku git:remote`) and run it in your terminal within your
  backend repo.

+ Push to Heroku: `git push heroku main`.

  + You may get an error message with `Failed to install gems via Bundler` in
    red. Above this, you might see a message that: 'Your bundle only supports
    platforms \["<some-platform>"] but your local platform is x86_64-linux'.

  + If you get this error, run the following command:

    ```sh
    bundle lock --add-platform x86_64-linux
    ```

    Then add all, commit, and run `git push heroku main` again.

  + You also might see a yellow `###### WARNING` letting you know that `There is
    a more recent Ruby version available for you to use.` Just ignore this for
    now. (Updating your Ruby version sounds simple, but it will likely take more
    time than you want to give it right now.)

  + If the process fails to complete successfully, read the error messages
    carefully and adjust accordingly. Ask an instructor if you get stuck.

### Step 5: Seeding your production database

Remember that your development database and production database are distinct.
The `release` command in your __Procfile__ took care of migrating your database,
but you might want to seed it too, at least after this initial push. To do
so, you will need access to the command line of your production app. This is
made easy by the Heroku CLI. To run a command in your app's production
environment on Heroku's computers, just start the command with `heroku run`.

+ Run the following command in your terminal to seed your database:

  ```sh
  heroku run rails db:seed
  ```

+ Test that the seed was successful:

  ```sh
  heroku console
  ```

  This will open your production Rails console using the command from your
  __Procfile__. (You could also use `heroku run rails c`.) Inside the console,
  run `User.all`. You should see the users you seeded to your production
  database!

### Step 6: See your site!

Once you've finished the above steps, run `heroku open` to open
`https://<your-app-name>.herokuapp.com` in a browser and see your live app!

If you see an `Application Error` or are experiencing different behavior than
what you see in your local environment, check the logs by running:

```bash
heroku logs
```

If you want Heroku to output the logs to your terminal continuously, run

```bash
heroku logs --tail
```

The logs may give you clues as to why you are experiencing errors or different
behavior.

### Wrapping up

If you made it this far, Congratulations! You've created a production-ready,
dynamic, full-stack website that can be securely accessed anywhere in the world!
Give yourself a pat on the back. You're a web developer!

[Heroku]: https://www.heroku.com/
[procfiles]: https://devcenter.heroku.com/articles/procfile
[heroku-signup]: https://signup.heroku.com/
[ruby-buildpack]: https://devcenter.heroku.com/articles/ruby-support
[dashboard]: https://dashboard.heroku.com/
[rails-credentials]: https://guides.rubyonrails.org/security.html#custom-credentials
[heroku-cli]: https://devcenter.heroku.com/articles/heroku-command-line
[config-vars]: https://devcenter.heroku.com/articles/config-vars
[`http:localhost:5000`]: http:localhost:5000
