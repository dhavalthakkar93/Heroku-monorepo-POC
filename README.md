# Monorepo and Heroku Pipeline with Review Apps

Imagine you have a single code base which has multiple application within it (here 3 applications Nginx, Node.JS and Ruby application), or you are following [Google's Monorepo](https://en.wikipedia.org/wiki/Monorepo) structure.

In that case how do you manage that with Heroku (Incl [pipeline](https://devcenter.heroku.com/articles/pipelines) and [Review Apps](https://devcenter.heroku.com/articles/github-integration-review-apps)?

# Usage 

We will use [heroku-buildpack-monorepo](https://github.com/lstoll/heroku-buildpack-monorepo) which uses [
heroku-buildpack-multi-procfile](https://github.com/heroku/heroku-buildpack-multi-procfile), This POC has 3 applications (Nginx Reverse Proxy, [Node.JS getting started application](https://github.com/heroku/node-js-getting-started), [Ruby getting started application](https://github.com/heroku/ruby-getting-started)): 

#### Create 3 Heroku applications

- `heroku create -a mono-nginx-app-stg`
- `heroku create -a mono-node-app-stg`
- `heroku create -a mono-ruby-app-stg`

#### Set relative path of application root directory (Where is Procfile) in application `APP_BASE` config var

- `heroku config:set -a mono-nginx-app-stg APP_BASE=nginx-app/`
- `heroku config:set -a mono-node-app-stg APP_BASE=node-app/`
- `heroku config:set -a mono-ruby-app-stg APP_BASE=ruby-app/`

#### Add Buildpack

- `heroku buildpacks:add -i 1 -a mono-nginx-app-stg https://github.com/lstoll/heroku-buildpack-monorepo`
- `heroku buildpacks:add -i 1 -a mono-node-app-stg https://github.com/lstoll/heroku-buildpack-monorepo`
- `heroku buildpacks:add -i 1 -a mono-ruby-app-stg https://github.com/lstoll/heroku-buildpack-monorepo`

#### Deploy application

You can either use [Github integration (Heroku Github Deploys)](https://devcenter.heroku.com/articles/github-integration) or you can push code to Heroku git using following commands:

- `git push git@heroku.com:mono-nginx-app-stg master`
- `git push git@heroku.com:mono-node-app-stg master`
- `git push git@heroku.com:mono-ruby-app-stg master`

#### What is the magic?

When application builds, buildpack (heroku-buildpack-monorepo) will copy `Procfile` from directory defined in `APP_BASE` config vars into `/app` directory of Heoku application.


# Usage with Heroku Pipeline

#### Create Pipeline

You can create pipeline from Heroku dashboard or using CLI command, Check more details [here](https://devcenter.heroku.com/articles/pipelines#creating-pipelines).

- `heroku pipelines:create -a mono-nginx-app-stg`

```
? Pipeline name: mono-pipeline
? Stage of mono-nginx-app-stg: staging
Creating example-pipeline pipeline... done
Adding mono-nginx-app-stg to mono-pipeline pipeline as staging... done
```

Note that the command must specify an existing app that you want to add to the pipeline.

The CLI prompts you to specify a name for the pipeline and a stage for the app you’re adding to it (`development`, `staging`, or `production`)

You can connect your diffrent branch to diffrent environments of pipeline and can configure the automatic deploys.

As we already set config var so when new build take place buildpack will automatically identify which code to deploy in which applicaion.

**Note**: Only **builds** will set the proper Procfile, promoting a slug downstream will not trigger a build, and therefore will not look at the environment variable and act accordingly. Make sure that the proper Procfile is referenced all the way upstream to the first stage that builds.

# Review Apps

To get monorepo work with Review Apps is bit tricky as there is no way to configure `app.json` for multiple directories, only one `app.json` schema is allowed, you can specify all the required buildpack and addons for every applications into one `app.json` schema, check [example app.json](https://github.com/dhavalthakkar93/Heroku-monorepo-POC/blob/master/app.json).

**Now question is how to handle multiple buildpack for each codebase?** (because If you don't have proper configuration files (like packge.json, Gemfile, requirements.txt) in your project directory language buildpack will throw error at build time).

So in this example we used 2 language specific buildpacks Node.JS (Requires package.json) and Ruby (Requires Gemfile), To handle this,  added [package.json](https://github.com/dhavalthakkar93/Heroku-monorepo-POC/blob/master/nginx-app/package.json) and [Gemfile](https://github.com/dhavalthakkar93/Heroku-monorepo-POC/blob/master/nginx-app/Gemfile) (Without any configurations) in `nginx-app`, added [Gemfile](https://github.com/dhavalthakkar93/Heroku-monorepo-POC/blob/master/node-app/Gemfile) (Without any configurations) in `node-app` and added [package.json](https://github.com/dhavalthakkar93/Heroku-monorepo-POC/blob/master/ruby-app/package.json) (without any configuratios) in `ruby-app`.

**Note**: Don't forgot to generate Gemfile.lock (It can be done by `bundle install`).

#### How to handle Pull Requests?

So this is tricky part, you need to add or update `APP_BASE` config var at the [pipeline's setting](https://devcenter.heroku.com/articles/github-integration-review-apps#sensitive-config-vars) before creating Review App, which will be injected into the review app when it is created and buildpack will identify which codebase to consider.

**Note**: You need to specify `APP_BASE` as [required](https://devcenter.heroku.com/articles/app-json-schema#env) to true (Which will indicate config var is required for app to function) in `app.json` file.


# Author

Dhaval Thakkar (dthakkar@heroku.com)

# Special Thanks
Special thanks to `Andrew Gwozdziewycz`, `Cyril David` and `Lincoln Stoll` for creating awesome buildpack ([heroku-buildpack-monorepo](https://github.com/lstoll/heroku-buildpack-monorepo)) to handle monorepo structure on Heroku.

