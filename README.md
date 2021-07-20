# rails_env

## Concept

* DeviseTokenAuth
* Active Storage, AWS S3
* Action Mailer, AWS SES
* RSpec
* Whenever
* Delayed Job
* TODO Deploy EC2
* TODO CertBot
* TODO `$ echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf && sudo sysctl -p`


## Usage    

### Create VM


1. `git clone git@github.com:hepeu/rails_env.git project_name`
1. `cd project_name`
1. `git checkout -b project_name`
1. `emacs playbook_common_vars.yml`
1. `git commit -a -m "Update library versions"`
1. `git push origin project_name`
1. `vagrant up`
1. `vagrant reload`
1. `vagrant ssh`


### Deploy to EC2 instances

1. `emacs playbook_common_vars.yml`
1. `emacs playbook_ec2.yml`
1. `ansible-playbook -i hosts.ec2 playbook_ec2.yml` 
1. `ssh server`
1. `sudo certbot --nginx`


## Ruby on Rails

### Create Project

1. `rails new project_name -d mysql --webpack --skip-webpack-install --skip-bundle`


### Gems

1. `emacs Gemfile`

    ```ruby
    source 'https://rubygems.org'
    git_source(:github) { |repo| "https://github.com/#{repo}.git" 

    ruby '3.0.1'

    gem 'rails', '~> 6.1.3.2'
    gem 'mysql2', '~> 0.5'
    gem 'puma', '~> 5.0'
    gem 'sass-rails', '>= 6'
    gem 'webpacker', '~> 5.0'
    gem 'image_processing', '~> 1.2'
    gem 'bootsnap', '>= 1.4.2', require: false
    gem 'dotenv-rails'
    gem 'devise_token_auth'
    gem 'rails-i18n'
    gem 'aws-sdk-rails' 
    gem 'aws-sdk-s3'
    gem 'delayed_job_active_record'
    gem 'daemons'
    gem 'whenever', require: false
    gem 'kaminari'

    group :development, :test do
      gem 'factory_bot_rails'
    end
    
    group :development do
      gem 'web-console', '>= 4.1.0'
      gem 'rack-mini-profiler', '~> 2.0'
      gem 'listen', '~> 3.3'
      gem 'spring'
      gem 'spring-commands-rspec'
    end

    group :test do
      gem 'capybara', '>= 3.26'
      gem 'selenium-webdriver'
      gem 'webdrivers'
      gem 'rspec-rails'
      gem 'rexml'     
    end
    ```

1. bundle install


### Config - .env

1. `emacs .env.vagrant`

    ```
    RE_DB_NAME=project_name 
    RE_DB_NAME_DEV=project_name
    RE_DB_NAME_TEST=project_name_test
    RE_DB_USERNAME=root
    RE_DB_PASSWORD=mymymymymysql
    RE_DB_HOSTNAME=localhost
    RE_DB_PORT=3306
    RE_STORAGE=local
    RE_PROTOCOL=http
    RE_HOST=localhost
    RE_PORT=3000
    ```
   
1. `ln -s .env.vagrant .env`   


### Config - Application

1. `emacs config/application.rb`
    ```ruby
    require_relative "boot"
    require "rails/all"
    Bundler.require(*Rails.groups)

    module ProjectName
      class Application < Rails::Application
        config.load_defaults 6.1
        config.i18n.default_locale = :ja
        config.time_zone = 'Tokyo'
        config.logger = Logger.new("log/project_name.log", 10, 5 * 1024 * 1024)
        config.active_job.queue_adapter = :delayed_job
        config.action_mailer.delivery_method = :ses
        config.action_mailer.default_url_options = {
          protocol: ENV['RE_PROTOCOL'],
          host: ENV['RE_HOST'],
          port: ENV['RE_PORT']
        }
        config.active_storage.service = ENV['RE_STORAGE'].to_sym
        routes.default_url_options = {
          protocol: ENV['RE_PROTOCOL'],
          host: ENV['RE_HOST'],
          port: ENV['RE_PORT']
        }
        config.generators do |g|
          g.test_framework :rspec,
                           fixtures: false,
                           view_specs: false,
                           helper_specs: false,
                           routing_specs: false
        end
      end
    end
    ```


### Config - Database

1. `emacs config/database.yml`

    ```yaml
    default: &default
      adapter: mysql2
      charset: utf8mb4
      encoding: utf8mb4
      collation: utf8mb4_unicode_ci
      pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

    development:
      <<: *default
      database: <%= ENV['RE_DB_NAME_DEV'] %>
      username: <%= ENV['RE_DB_USERNAME'] %>
      password: <%= ENV['RE_DB_PASSWORD'] %>
      host: <%= ENV['RE_DB_HOSTNAME'] %>
      port: <%= ENV['RE_DB_PORT'] %>
   
    test:
      <<: *default
      database: <%= ENV['RE_DB_NAME_TEST'] %>
      username: <%= ENV['RE_DB_USERNAME'] %>
      password: <%= ENV['RE_DB_PASSWORD'] %>
      host: <%= ENV['RE_DB_HOSTNAME'] %>
      port: <%= ENV['RE_DB_PORT'] %>
   
    production:
      <<: *default
      database: <%= ENV['RE_DB_NAME'] %>
      username: <%= ENV['RE_DB_USERNAME'] %>
      password: <%= ENV['RE_DB_PASSWORD'] %>
      host: <%= ENV['RE_DB_HOSTNAME'] %>
      port: <%= ENV['RE_DB_PORT'] %>
    ```

1. `rails db:create`


### Config - Puma

1. `emacs config/puma.rb`

    ```ruby
    # Comment
    # port        ENV.fetch("PORT") { 3000 }
    # Add
    port '3000', '0.0.0.0'
    ```


### Config - Keys

1. `rails credentials:edit`
    ```yaml
    s3:
      access_key_id: "XXX..."
      secret_access_key: "XXX..."

    s3_hepeu:
      access_key_id: "XXX..."
      secret_access_key: "XXX..."

    ses:
      access_key_id: "XXX..."
      secret_access_key: "XXX..."

    # Used as the base secret for all MessageVerifiers in Rails, including the one protecting cookies.
    secret_key_base: XXX...
    ```


### AWS SES

1. `emacs config/initializers/ses.rb`
    ```ruby
    credentials = Aws::Credentials.new(Rails.application.credentials.dig(:ses, :access_key_id),
                                       Rails.application.credentials.dig(:ses, :secret_access_key))
    Aws::Rails.add_action_mailer_delivery_method(:ses,
                                                 credentials: credentials,
                                                 region: 'us-east-1')   
    ```

1. `emacs app/mailers/application_mailer.rb`
   ```ruby
   default from: 'noreply@hepeu.com'
   ```


### Active Storage, AWS S3

1. Create buckets named project-name-s3 and project-name-s3-hepeu in AWS Console.

1. `rails active_storage:install`

1. `rails db:migrate`

1. `emacs config/storage.yml`
    ```yaml
    test:
      service: Disk
      root: <%= Rails.root.join("tmp/storage") %>

    local:
      service: Disk
      root: <%= Rails.root.join("storage") %>
    
    s3:
      service: S3
      access_key_id: <%= Rails.application.credentials.dig(:s3, :access_key_id) %>
      secret_access_key: <%= Rails.application.credentials.dig(:s3, :secret_access_key) %>
      region: ap-northeast-1
      bucket: project-name-s3
     
    s3_hepeu:
      service: S3
      access_key_id: <%= Rails.application.credentials.dig(:s3_hepeu, :access_key_id) %>
      secret_access_key: <%= Rails.application.credentials.dig(:s3_hepeu, :secret_access_key) %>
      region: ap-northeast-1
      bucket: project-name-s3-hepeu
    ```


### Devise Token Auth

1. `rails generate devise:install`

1. `rails generate devise_token_auth:install User auth`

1. `emacs app/models/user.rb`
   ```ruby
   class User < ActiveRecord::Base
     #:omniauthable, :registerable, :rememberable, :timeoutable
     devise :database_authenticatable,
            :confirmable,
            :recoverable,
            :validatable,
            :trackable,
            :confirmable,
            :lockable
     include DeviseTokenAuth::Concerns::User
   end
   ```

1. `emacs db/migrate/20..._devise_token_auth_create_users.rb`
    ```ruby
    class DeviseTokenAuthCreateUsers < ActiveRecord::Migration[5.2]
      def change
        create_table(:users) do |t|
          ## Required
          t.string :provider, :null => false, :default => "email"
          t.string :uid, :null => false, :default => ""

          ## Database authenticatable
          t.string :encrypted_password, :null => false, :default => ""
 
          ## Recoverable
          t.string   :reset_password_token
          t.datetime :reset_password_sent_at
          t.boolean  :allow_password_change, :default => false

          ## Rememberable
          # t.datetime :remember_created_at
 
          ## Trackable
          t.integer  :sign_in_count, :default => 0, :null => false
          t.datetime :current_sign_in_at
          t.datetime :last_sign_in_at
          t.string   :current_sign_in_ip
          t.string   :last_sign_in_ip

          ## Confirmable
          t.string   :confirmation_token
          t.datetime :confirmed_at
          t.datetime :confirmation_sent_at
          t.string   :unconfirmed_email # Only if using reconfirmable

          ## Lockable
          t.integer  :failed_attempts, :default => 0, :null => false # Only if lock strategy is :failed_attempts
          t.string   :unlock_token # Only if unlock strategy is :email or :both
          t.datetime :locked_at
 
          ## User Info
          t.string :name
          t.string :email
          t.string :role

          ## Tokens
          t.text :tokens
 
          t.timestamps
        end

        add_index :users, :email,                unique: true
        add_index :users, [:uid, :provider],     unique: true
        add_index :users, :reset_password_token, unique: true
        add_index :users, :confirmation_token,   unique: true
        add_index :users, :unlock_token,         unique: true
      end
    end
    ```

1. `emacs config/initializers/devise.rb`
    ```ruby
    Devise.setup do |config|
      config.mailer_sender = 'noreply@hepeu.com'
    end
    ```

1. `emacs config/initializers/devise_token_auth.rb`
   ```ruby
   config.change_headers_on_each_request = false
   config.token_lifespan = 2.weeks
   ```

1. `rails db:migrate`


### RSpec

1. `rails generate rspec:install`

1. `emacs .rspec`
    ```
    --require spec_helper
    --format documentation
    ```
    
1. `emacs spec/rails_helper.rb`
    ```ruby
    RSpec.configure do |config|
      ...
      # ADD
      config.include Devise::Test::ControllerHelpers, type: :controller 
    ```
    
1. `bundle exec spring binstub rspec`

1. `bin/rspec`

1. `rm -rf test`


### HomeController

1. `rails generate controller Home index`

1. `emacs config/routes.rb`
    ```ruby
    root to: 'home#index'
    ```

1. `rm app/assets/stylesheets/home.scss`


### Install webpacker

1. `rails webpacker:install`
   

## Break

1. `rails s`

1. Access `http://localhost:3000` from your host browser.

1. `emacs .gitignore`
    ```gitignore
    # Comment
    # /config/master.key
    # ADD
    .env
    .precompile_version
    ```

1. `git add .`

1. `git commit -a -m "Initial import"`


### Vue, Vuetify

1. `rails webpacker:install:vue`

1. `yarn add vuex vue-router axios vuetify`

1. `rm app/javascript/app.vue`

1. `rm app/javascript/packs/*`

1. `emacs app/views/layouts/application.html.erb`
    ```erb
    <!-- remove , 'data-turbolinks-track': 'reload' -->
    <%= stylesheet_link_tag 'application', media: 'all', 'data-turbolinks-track': 'reload' %>
    <!-- remove line -->
    <%= javascript_pack_tag 'application', 'data-turbolinks-track': 'reload' %>
    ````

### Console::HomeController

1. `rails generate controller Console::Home index`

1. `emacs config/routes.rb`
    ```ruby
    # ADD
    namespace :console do
      get '/', to: 'home#index'
    end
    ```
    
1. `emacs app/views/console/home/index.html.erb`

   - Remove all lines.

1. `mkdir -p app/views/layouts/console`

1. `emacs app/views/layouts/console/home.html.erb`
    ```erb
    <!DOCTYPE html>
    <html>
      <head>
        <title>project_name</title>
        <%= csrf_meta_tags %>
        <%= csp_meta_tag %>
        <link href="https://fonts.googleapis.com/css?family=Roboto:100,300,400,500,700,900" rel="stylesheet">
        <link href="https://fonts.googleapis.com/icon?family=Material+Icons" rel="stylesheet">
        <link href="https://cdn.jsdelivr.net/npm/@mdi/font@4.x/css/materialdesignicons.min.css" rel="stylesheet">
        <meta name="viewport" content="width=device-width,initial-scale=1">
      </head>
      <body>
        <%= yield %>
        <%= javascript_pack_tag 'console/app_vue' %>
        <%= stylesheet_pack_tag 'console/app_vue' %>
      </body>
    </html>
    ```

1. `mkdir -p app/javascript/packs/console`

1. `emacs app/javascript/packs/console/app_vue.js`
    ```javascript
    import Vue from 'vue'
    import App from './app.vue'
    
    document.addEventListener('DOMContentLoaded', () => {
      document.body.appendChild(document.createElement('project_name'))
      const app = new Vue(App).$mount('project_name')
    })
    ```

1. `emacs app/javascript/packs/console/app.vue`
    ```vue
    <template>
      <v-app>
        <router-view></router-view>
      </v-app>
    </template>
        
    <script>
    import Vue from 'vue'
    import VueRouter from 'vue-router'
    import Vuex from 'vuex'
    import Vuetify from 'vuetify'
    import 'vuetify/dist/vuetify.min.css'
      
    import Home from './home.vue'
        
    Vue.use(VueRouter)
    Vue.use(Vuex)
    Vue.use(Vuetify)
        
    const vuetify = new Vuetify()
       
    let router = new VueRouter({
      routes: [
        { path: '/', component: Home }
      ]
    })
                
    let store = new Vuex.Store({
    })
                
    export default {
      router,
      store,
      vuetify
    }
    </script>
                      
    <style>
    body {
      width: 100%;
      height: 100%;
    }
    </style>
    ```

1. `emacs app/javascript/packs/console/home.vue`
    ```vue
    <template>
      <h1>Hello, world</h1>
    </template>
    ```


### Info Mailer

1. `rails generate mailer InfoMailer`

1. `emacs app/mailers/info_mailer.rb`
    ```ruby
    class InfoMailer < ApplicationMailer
      def info
        @message = params[:message]
        mail(to: params[:to], subject: params[:subject])
      end
    end
    ```
    
1. `emacs app/views/info_mailer/info.html.erb`
    ```erb
    <p>
      <%= @message %>
    </p>
    ```
    
1. `emacs app/views/info_mailer/info.text.erb`
    ```erb
    <%= @message %> 
    ```
    

### Delayed Job

1. `rails generate delayed_job:active_record`

1. `rails db:migrate`

1. `rails generate job DelayedJobStart`

1. `emacs app/jobs/delayed_job_start_job.rb`
    ```ruby
    class DelayedJobStartJob < ApplicationJob
      queue_as :default
        
      def perform(*args)
        bin = Rails.root / 'bin' / 'delayed_job'
        status = `#{bin} status`
        return if status.include?('pid')
        `#{bin} restart`

        to = 'takeshi.shoji@hepeu.com'
        subject = '[INFO] Delayed job restarted'
        message = "#{Time.zone.now}: Delayed job restarted"
        InfoMailer.with(to: to, subject: subject, message: message).info.deliver_later
      end
    end
    ```


### Whenever

1. `bundle exec wheneverize .`

1. `emacs config/schedule.rb`
    ```ruby
    require 'active_support/core_ext/time'

    def jst2utc(time)
      Time.zone = 'Asia/Tokyo'
      Time.zone.parse(time).localtime($system_utc_offset)
    end

    set :path, "#{ENV['HOME']}/project_name"
    set :job_template, "/bin/bash -l -i -c ':job'"

    if ENV['USER'] == 'vagrant' then
      set :output, "#{ENV['HOME']}/project_name/log/cron.log"
      set :environment, "development"
      every 5.minute do
        runner "DelayedJobStartJob.perform_now"
      end
      # every 60.minute do
      # runner "SomeJob.perform_now"
      # end
    else
      set :environment, "production"
      every 30.minute do
        runner "DelayedJobStartJob.perform_now"
      end
      # every 1.day, at: jst2utc('0:30 am') do
      #   runner "SomeJob.perform_now"
      # end
    end
    ```

1. `bundle exec whenever`

1. `bundle exec whenever --upcate-crontab`

1. `crontab -l`


### ./run.sh

1.  `emacs ./run.sh`
    ```sh
    export RAILS_ENV=production
    export RAILS_SERVE_STATIC_FILES=true
    
    export PATH="/home/ubuntu/.rbenv/bin:$PATH"
    export PATH="/home/ubuntu/.rbenv/shims:$PATH"
    eval "$(rbenv init -)"
    
    bundle install
    bundle exec rails r DelayedJobStartJob.perform_now

    PRECOMPILE_VERSION=`git rev-parse HEAD`
    if [ ! -f '.precompile_version' ] || [ ! `cat .precompile_version` = $PRECOMPILE_VERSION ]; then
        bundle exec rails assets:precompile
        bundle exec rails assets:clean
        echo $PRECOMPILE_VERSION > '.precompile_version'
    fi

    bundle exec rails s
    ```                                                             
# suraud
