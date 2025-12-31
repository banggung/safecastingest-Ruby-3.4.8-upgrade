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
