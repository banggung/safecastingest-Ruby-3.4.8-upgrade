# safecastingest-Ruby-3.4.8-upgrade

Edit `Gemfile`:
```diff
source 'https://rubygems.org' do
- # Puma with Ruby 2.6 running on 64bit Amazon Linux/2.11.7
- ruby '2.6.6'
+  # Ruby 3.4 running on 64bit Amazon Linux 2023
+  ruby '3.4.8'

  gem 'puma'
  gem 'rake'

  # api
  gem 'grape'
  gem 'grape_logging'
- gem 'actionview', '>= 5.0.7.2'
- gem 'active_model_serializers'
+  gem 'actionview', '>= 7.0'
+  gem 'active_model_serializers', '~> 0.10.14'

  # database/orms
  gem 'pg'
  gem 'otr-activerecord'
- gem 'activerecord-postgis-adapter', '>= 5.2.2'
+  gem 'activerecord-postgis-adapter', '>= 7.0'

  # data pipeline
  gem 'aws-sdk-sns', '~> 1'
  gem 'aws-sdk-sqs', '~> 1'
  gem 'aws-sdk-s3', '~> 1'
- gem 'elasticsearch', '~> 5.0'
+  gem 'elasticsearch', '~> 8.0'

  gem 'dotenv'
  gem 'newrelic_rpm'
  gem 'rison'
  gem 'thor'
  gem 'faraday'

  # elasticbeanstalk rake task
  gem 'aws-sdk-elasticbeanstalk'

+  # Ruby 3.x compatibility
+  gem 'bigdecimal'
+  gem 'mutex_m'
+  gem 'drb'
+  gem 'base64'
+  gem 'ostruct'
+
  group :development do
    gem 'rerun'
  end

  group :test do
    gem 'database_cleaner'
    gem 'rack-test'
    gem 'rspec'
    gem 'rspec-json_matcher'
    gem 'rspec_junit_formatter'
    gem 'shoulda-matchers'
  end

  group :development, :test do
    gem 'annotate'
    gem 'factory_bot'
    gem 'pry'
  end
end
```


Edit `.ebextensions/db.config`:

```diff
-commands:
-  upgrade_postgres:
-    command: |
-      /usr/bin/yum -y remove 'postgres*'
-      /usr/bin/yum -y install \
-        'https://yum.postgresql.org/11/redhat/rhel-6-x86_64/postgresql11-libs-11.5-1PGDG.rhel6.x86_64.rpm' \
-        'https://yum.postgresql.org/11/redhat/rhel-6-x86_64/postgresql11-11.5-1PGDG.rhel6.x86_64.rpm' \
-        'https://yum.postgresql.org/11/redhat/rhel-6-x86_64/postgresql11-devel-11.5-1PGDG.rhel6.x86_64.rpm'
-      /bin/ln -sf /usr/pgsql-11/bin/pg_config /usr/bin/pg_config
+packages:
+  yum:
+    postgresql15: []
+    libpq-devel: []

 files:
   "/etc/profile.d/z_psql.sh":
     content: |
       export PGHOST="${DATABASE_HOST}"
       export PGPORT=5432
       export PGDATABASE=safecast
       export PGUSER=safecast

       cat > ~/.pgpass <<EOF
       ${DATABASE_HOST}:5432:safecast:safecast:${DATABASE_PASSWORD}
       EOF
       chmod 600 ~/.pgpass
```

Edit `.ebextensions/00_base.config`:

```diff
 packages:
   yum:
     jq: []
-
-commands:
-  parameters_cache:
-    command: |
-      /opt/aws/bin/cfn-get-metadata --region `{"Ref": "AWS::Region"}` --stack `{"Ref": "AWS::StackName"}` \
-        --resource AWSEBBeanstalkMetadata --key AWS::ElasticBeanstalk::Ext |
-        jq .Parameters > /etc/elasticbeanstalk/parameters-cache
```

Edit `.ebextensions/50-post-deploy.config`:

```diff
-commands:
-  create_post_dir:
-    command: "mkdir -p /opt/elasticbeanstalk/hooks/appdeploy/post"
 files:
-  "/opt/elasticbeanstalk/hooks/appdeploy/post/50_restart_workers.sh":
+  "/opt/elasticbeanstalk/hooks/postdeploy/50_restart_workers.sh":
     mode: "000755"
     owner: root
     group: root
     content: |
       #!/usr/bin/env bash
-      
-      source /opt/elasticbeanstalk/support/envvars
+
+      source /opt/elasticbeanstalk/deployment/env

-      if grep -q ELASTIC_CLOUD_INPUT_QUEUE_URL /etc/elasticbeanstalk/parameters-cache; then
-        if status worker_elastic_cloud | grep -q running; then
-          logger -t worker_elastic_cloud 'Restarting worker after deploy.'
-          restart worker_elastic_cloud
-        else
-          logger -t worker_elastic_cloud 'Starting worker after deploy.'
-          start worker_elastic_cloud
-        fi
-      else
-        logger -t worker_elastic_cloud 'This does not appear to be a worker node. Setting worker to manual after deploy.'
-        echo manual > /etc/init/worker_elastic_cloud.override
+      # Check if this is a worker environment
+      if [[ -n "${ELASTIC_CLOUD_INPUT_QUEUE_URL}" ]]; then
+        logger -t worker_elastic_cloud 'Restarting worker after deploy.'
+        systemctl restart worker_elastic_cloud || systemctl start worker_elastic_cloud
       fi
-      if grep -q S3_RAW_INPUT_QUEUE_URL /etc/elasticbeanstalk/parameters-cache; then
-        if status worker_s3_raw | grep -q running; then
-          logger -t worker_s3_raw 'Restarting worker after deploy.'
-          restart worker_s3_raw
-        else
-          logger -t worker_s3_raw 'Starting worker after deploy.'
-          start worker_s3_raw
-        fi
-      else
-        logger -t worker_s3_raw 'This does not appear to be a worker node. Setting worker to manual after deploy.'
-        echo manual > /etc/init/worker_s3_raw.override
+
+      if [[ -n "${S3_RAW_INPUT_QUEUE_URL}" ]]; then
+        logger -t worker_s3_raw 'Restarting worker after deploy.'
+        systemctl restart worker_s3_raw || systemctl start worker_s3_raw
       fi
```

Edit `.ebextensions/workers.config`:

```diff
 files:
-  "/etc/init/worker_s3_raw.conf":
+  "/etc/systemd/system/worker_s3_raw.service":
+    mode: "000644"
+    owner: root
+    group: root
     content: |
-      description "Start worker for writing raw points to S3"
-
-      start on runlevel [2345]
-      stop on runlevel [!2345]
-
-      respawn
-      respawn limit unlimited
-
-      script
-        cd /var/app/current
-        sudo -u webapp bash -lc 'rake workers:s3_raw' | logger -t worker_s3_raw
-      end script
-
-  "/etc/init/worker_elastic_cloud.conf":
+      [Unit]
+      Description=Worker for writing raw points to S3
+      After=network.target
+
+      [Service]
+      Type=simple
+      User=webapp
+      WorkingDirectory=/var/app/current
+      ExecStart=/bin/bash -lc 'bundle exec rake workers:s3_raw'
+      Restart=always
+      RestartSec=5
+      StandardOutput=journal
+      StandardError=journal
+      SyslogIdentifier=worker_s3_raw
+      EnvironmentFile=/opt/elasticbeanstalk/deployment/env
+
+      [Install]
+      WantedBy=multi-user.target
+
+  "/etc/systemd/system/worker_elastic_cloud.service":
+    mode: "000644"
+    owner: root
+    group: root
     content: |
-      description "Start worker for writing points to elastic cloud"
-
-      start on runlevel [2345]
-      stop on runlevel [!2345]
-
-      respawn
-      respawn limit unlimited
-
-      script
-        cd /var/app/current
-        sudo -u webapp bash -lc 'rake workers:elastic_cloud' | logger -t worker_elastic_cloud
-      end script
+      [Unit]
+      Description=Worker for writing points to elastic cloud
+      After=network.target
+
+      [Service]
+      Type=simple
+      User=webapp
+      WorkingDirectory=/var/app/current
+      ExecStart=/bin/bash -lc 'bundle exec rake workers:elastic_cloud'
+      Restart=always
+      RestartSec=5
+      StandardOutput=journal
+      StandardError=journal
+      SyslogIdentifier=worker_elastic_cloud
+      EnvironmentFile=/opt/elasticbeanstalk/deployment/env
+
+      [Install]
+      WantedBy=multi-user.target

 commands:
-  01_match_nginx_timeout_to_sqs_timeout:
-    command: |
-      VISIBILITY_TIMEOUT=$(jq -r .AWSEBVisibilityTimeout /etc/elasticbeanstalk/parameters-cache)
-      if [[ "${VISIBILITY_TIMEOUT}" != "null" ]]; then
-        echo "proxy_read_timeout ${VISIBILITY_TIMEOUT}s;" > /etc/nginx/conf.d/worker.conf
-        service nginx restart
-      fi
+  01_reload_systemd:
+    command: systemctl daemon-reload
```

Edit `application.rb`:
```diff
 require_relative 'config/environment'

 # Database connection
 OTR::ActiveRecord.configure_from_file!(
   Config.root.join('config', 'database.yml')
 )

+# Explicitly establish connection for ActiveRecord 7.x
+ActiveRecord::Base.establish_connection(Config.env.to_sym)
+
 # Load application
 [
   %w(app lib *.rb),
   %w(config initializers *.rb),
   %w(config initializers ** *.rb),
   %w(app models ** *.rb),
   %w(app models *.rb),
   %w(app serializers ** *.rb),
   %w(app serializers *.rb),
   %w(app api ** *.rb),
   %w(app api *.rb),
   %w(app workers batch.rb),
   %w(app workers *.rb)
 ].each do |pattern|
   Dir.glob(Config.root.join(*pattern)).each { |file| require file }
 end
-
-::ActiveRecord::Base.schema_format = :sql
-::ActiveRecord::Base.dump_schemas = :all
```

Delete Gemfile.lock
```bash
$ rm Gemfile.lock
```

bundle install
```
$ bundle install
bundle install
Fetching gem metadata from https://rubygems.org/..........
Resolving dependencies...
Fetching rake 13.3.1
Installing rake 13.3.1
Fetching base64 0.3.0
Fetching benchmark 0.5.0
Installing base64 0.3.0
Installing benchmark 0.5.0
Fetching bigdecimal 4.0.1
Fetching concurrent-ruby 1.3.6
Installing bigdecimal 4.0.1 with native extensions
Installing concurrent-ruby 1.3.6
Fetching connection_pool 3.0.2
Installing connection_pool 3.0.2
Fetching drb 2.2.3
Installing drb 2.2.3
Fetching logger 1.7.0
Installing logger 1.7.0
Fetching prism 1.7.0
Installing prism 1.7.0 with native extensions
Fetching securerandom 0.4.1
Installing securerandom 0.4.1
Fetching builder 3.3.0
Installing builder 3.3.0
Fetching cgi 0.5.1
Installing cgi 0.5.1 with native extensions
Fetching erubi 1.13.1
Installing erubi 1.13.1
Fetching crass 1.0.6
Installing crass 1.0.6
Fetching rack 3.2.4
Installing rack 3.2.4
Fetching useragent 0.16.11
Installing useragent 0.16.11
Fetching jsonapi-renderer 0.2.2
Installing jsonapi-renderer 0.2.2
Fetching timeout 0.6.0
Installing timeout 0.6.0
Fetching rgeo 3.0.1
Installing rgeo 3.0.1 with native extensions
Fetching amazing_print 2.0.0
Installing amazing_print 2.0.0
Fetching aws-eventstream 1.4.0
Installing aws-eventstream 1.4.0
Fetching aws-partitions 1.1200.0
Installing aws-partitions 1.1200.0
Fetching jmespath 1.6.2
Installing jmespath 1.6.2
Fetching coderay 1.1.3
Installing coderay 1.1.3
Fetching database_cleaner-core 2.0.1
Installing database_cleaner-core 2.0.1
Fetching diff-lcs 1.6.2
Installing diff-lcs 1.6.2
Fetching dotenv 3.2.0
Installing dotenv 3.2.0
Fetching zeitwerk 2.7.4
Installing zeitwerk 2.7.4
Fetching dry-inflector 1.2.0
Installing dry-inflector 1.2.0
Fetching uri 1.1.1
Installing uri 1.1.1
Fetching json 2.18.0
Installing json 2.18.0 with native extensions
Fetching multi_json 1.19.1
Installing multi_json 1.19.1
Fetching ffi 1.17.3 (aarch64-linux-gnu)
Installing ffi 1.17.3 (aarch64-linux-gnu)
Fetching ruby2_keywords 0.0.5
Installing ruby2_keywords 0.0.5
Fetching io-console 0.8.2
Installing io-console 0.8.2 with native extensions
Fetching rb-fsevent 0.11.2
Installing rb-fsevent 0.11.2
Fetching method_source 1.1.0
Installing method_source 1.1.0
Fetching newrelic_rpm 9.24.0
Installing newrelic_rpm 9.24.0
Fetching nio4r 2.7.5
Installing nio4r 2.7.5 with native extensions
Fetching ostruct 0.6.3
Installing ostruct 0.6.3
Fetching parslet 1.8.2
Installing parslet 1.8.2
Fetching pg 1.6.3 (aarch64-linux)
Installing pg 1.6.3 (aarch64-linux)
Fetching rspec-support 3.13.6
Installing rspec-support 3.13.6
Fetching thor 1.4.0
Installing thor 1.4.0
Fetching i18n 1.14.8
Installing i18n 1.14.8
Fetching tzinfo 2.0.6
Installing tzinfo 2.0.6
Fetching nokogiri 1.19.0 (aarch64-linux-gnu)
Installing nokogiri 1.19.0 (aarch64-linux-gnu)
Fetching rack-session 2.1.1
Installing rack-session 2.1.1
Fetching rack-test 2.2.0
Installing rack-test 2.2.0
Fetching aws-sigv4 1.12.1
Installing aws-sigv4 1.12.1
Fetching dry-core 1.2.0
Installing dry-core 1.2.0
Fetching net-http 0.9.1
Installing net-http 0.9.1
Fetching rspec-json_matcher 0.2.0
Installing rspec-json_matcher 0.2.0
Fetching elasticsearch-api 8.19.2
Installing elasticsearch-api 8.19.2
Fetching rb-inotify 0.11.1
Installing rb-inotify 0.11.1
Fetching mustermann 3.0.4
Installing mustermann 3.0.4
Fetching reline 0.6.3
Installing reline 0.6.3
Fetching puma 7.1.0
Installing puma 7.1.0 with native extensions
Fetching rison 2.1.0
Installing rison 2.1.0
Fetching rspec-core 3.13.6
Installing rspec-core 3.13.6
Fetching rspec-expectations 3.13.5
Installing rspec-expectations 3.13.5
Fetching rspec-mocks 3.13.7
Installing rspec-mocks 3.13.7
Fetching loofah 2.25.0
Installing loofah 2.25.0
Fetching aws-sdk-core 3.240.0
Installing aws-sdk-core 3.240.0
Fetching dry-configurable 1.3.0
Installing dry-configurable 1.3.0
Fetching dry-logic 1.6.0
Installing dry-logic 1.6.0
Fetching faraday-net_http 3.4.2
Installing faraday-net_http 3.4.2
Fetching listen 3.9.0
Installing listen 3.9.0
Fetching mustermann-grape 1.1.0
Installing mustermann-grape 1.1.0
Fetching pry 0.16.0
Installing pry 0.16.0
Fetching rspec_junit_formatter 0.6.0
Installing rspec_junit_formatter 0.6.0
Fetching rspec 3.13.2
Installing rspec 3.13.2
Fetching rails-html-sanitizer 1.6.2
Installing rails-html-sanitizer 1.6.2
Fetching aws-sdk-elasticbeanstalk 1.96.0
Installing aws-sdk-elasticbeanstalk 1.96.0
Fetching aws-sdk-kms 1.118.0
Installing aws-sdk-kms 1.118.0
Fetching aws-sdk-sns 1.109.0
Installing aws-sdk-sns 1.109.0
Fetching aws-sdk-sqs 1.107.0
Installing aws-sdk-sqs 1.107.0
Fetching dry-types 1.7.1
Installing dry-types 1.7.1
Fetching faraday 2.14.0
Installing faraday 2.14.0
Fetching rerun 0.14.0
Installing rerun 0.14.0
Fetching aws-sdk-s3 1.209.0
Installing aws-sdk-s3 1.209.0
Fetching elastic-transport 8.4.1
Installing elastic-transport 8.4.1
Fetching elasticsearch 8.19.2
Installing elasticsearch 8.19.2
Fetching minitest 6.0.1
Installing minitest 6.0.1
Fetching activesupport 7.2.3
Installing activesupport 7.2.3
Fetching rails-dom-testing 2.3.0
Fetching activemodel 7.2.3
Installing rails-dom-testing 2.3.0
Installing activemodel 7.2.3
Fetching case_transform 0.2
Installing case_transform 0.2
Fetching factory_bot 6.5.6
Installing factory_bot 6.5.6
Fetching grape 3.0.1
Installing grape 3.0.1
Fetching shoulda-matchers 7.0.1
Installing shoulda-matchers 7.0.1
Fetching actionview 7.2.3
Fetching activerecord 7.2.3
Installing actionview 7.2.3
Installing activerecord 7.2.3
Fetching grape_logging 3.0.0
Installing grape_logging 3.0.0
Fetching actionpack 7.2.3
Installing actionpack 7.2.3
Fetching active_model_serializers 0.10.16
Installing active_model_serializers 0.10.16
Fetching rgeo-activerecord 8.0.0
Installing rgeo-activerecord 8.0.0
Fetching annotate 3.2.0
Fetching database_cleaner-active_record 2.2.2
Installing annotate 3.2.0
Installing database_cleaner-active_record 2.2.2
Fetching otr-activerecord 2.6.0
Fetching activerecord-postgis-adapter 10.0.1
Installing otr-activerecord 2.6.0
Installing activerecord-postgis-adapter 10.0.1
Fetching database_cleaner 2.1.0
Installing database_cleaner 2.1.0
Bundle complete! 34 Gemfile dependencies, 105 gems now installed.
Use `bundle info [gemname]` to see where a bundled gem is installed.
2 installed gems you directly depend on are looking for funding.
  Run `bundle fund` for details
```

Check funding
```bash
$ bundle fund
* activerecord-postgis-adapter (10.0.1)
  Funding: https://opencollective.com/rgeo
* dotenv (3.2.0)
  Funding: https://github.com/sponsors/bkeepers
```



Old environment uses TCP:80 health check (just checks if port is open, not HTTP response).
For the new ALB-based environment, set health check to accept 404:

In EB Console for safecastingest-prd-010:
- Go to Configuration → Load balancer → Processes
- Edit default process
- Set Health check path: /
- Set HTTP code: 200-404

```bash
% aws elasticbeanstalk update-environment \
  --environment-name safecastingest-prd-010 \
  --option-settings \
    Namespace=aws:elasticbeanstalk:environment:process:default,OptionName=MatcherHTTPCode,Value=200-404 \
  --profile safecast --region us-west-2

{
    "EnvironmentName": "safecastingest-prd-010",
    "EnvironmentId": "e-xe3uxbdcqm",
    "ApplicationName": "ingest",
    "VersionLabel": "ingest_AL2023_Ruby3.4.8",
    "SolutionStackName": "64bit Amazon Linux 2023 v4.8.1 running Ruby 3.4",
    "PlatformArn": "arn:aws:elasticbeanstalk:us-west-2::platform/Ruby 3.4 running on 64bit Amazon Linux 2023/4.8.1",
    "EndpointURL": "awseb--AWSEB-wZ1r5DHHtXyc-1705929904.us-west-2.elb.amazonaws.com",
    "CNAME": "safecastingest-prd-010.saej7m7pg2.us-west-2.elasticbeanstalk.com",
    "DateCreated": "2025-12-31T09:14:21.057000+00:00",
    "DateUpdated": "2025-12-31T11:07:38.606000+00:00",
    "Status": "Updating",
    "AbortableOperationInProgress": true,
    "Health": "Grey",
    "Tier": {
        "Name": "WebServer",
        "Type": "Standard",
        "Version": "1.0"
    },
    "EnvironmentArn": "arn:aws:elasticbeanstalk:us-west-2:985752656544:environment/ingest/safecastingest-prd-010"
}
```

```bash
% aws elasticbeanstalk describe-environments --environment-names safecastingest-prd-010 --query "Environments[0].{Health:Health,HealthStatus:HealthStatus}" --profile safecast --region us-west-2
{
    "Health": "Green",
    "HealthStatus": "Ok"
}
```
