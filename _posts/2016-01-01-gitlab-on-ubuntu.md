---
layout: post
title: "GitLab on Ubuntu"
categories: GitLab Ubuntu
---

## About this document

This document will guide you through installing [GitLab](http://gitlab.com) on an Ubuntu-based system.
After successful installation the GitLab web interface will be accessible at
[https://danielflecken.de:444](https://danielflecken.de:444).
Since GitLab must be configured as the root application and we want to also serve other SSL-secured web applications from the same host, we are using the non-standard port 444 for the SSL-HTTP connection.

The installation was tested with:

- Ubuntu 14.04 LTS
- 64-bit machine (required for the GitLab CE Omnibus package)
- GitLab 8.0.5

All commands mentioned in this guide must be executed as user *root*.

## 1) Base installation

Install the packages GitLab depends on:
```
apt-get install curl openssh-server ca-certificates postfix
```

Add the GitLab package server and install the package:
```
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | bash
apt-get install gitlab-ce
```

## 2) Configuration

Place the SSL files inside GitLab's configuration direcory:

- Private key: */etc/gitlab/ssl/danielflecken.de.key*
- Public certificate: */etc/gitlab/ssl/danielflecken.de.crt*

> Note: The file names follow the GitLab naming conventions
> so they can be found by GitLab automatically
> (hostname plus suffixes *.key* and *.crt* respectively).
> If you already have the SSL files somewhere else on the server,
> you can create symbolic links to them,
> using the file names mentioned here as link names.

Make sure the private key can only be read by user *root*:
```
chmod u=rw,g=r,o= /etc/gitlab/ssl/danielflecken.de.key
```

Modify the GitLab base URL in the configuration file */etc/gitlab/gitlab.rb*:
```
external_url 'https://danielflecken.de:444'
```

> Note: GitLab will automatically serve its content
> from the port you specify in *external_url*.

Modify the configuration file to make GitLab delete old backups
after 7 days (604,800 seconds):
```
gitlab_rails['backup_keep_time'] = 604800
```

Auto-configure and start GitLab:
```
gitlab-ctl reconfigure
```

## 3) Automating backups

We will create daily backups of our GitLab installation
and the GitLab configuration files.
The directory holding the backups will be synchronized
into an [Amazon S3](https://aws.amazon.com/s3/) bucket.
For encrypted synchronization we will use [Duply](http://duply.net/),
a frontend for [Duplicity](http://duplicity.nongnu.org/).

GitLab has a pre-defined [Rake](http://rake.rubyforge.org) task
for creating backup archives of the GitLab data.
Define a new task in */opt/gitlab/embedded/service/gitlab-rails/lib/tasks/gitlab/backup_config.rake*
for archiving the GitLab configuration files:
```
namespace :gitlab do
  namespace :backup_config do

    desc "GitLab | Create a backup of the GitLab configuration"
    task create: :environment do
      pack
      remove_old
    end
    
    def pack
      warn_user_is_not_gitlab
      configure_cron_mode
      
      tar_file = "#{Time.now.to_i}_gitlab_config_backup.tar"
      
      Dir.chdir(Gitlab.config.backup.path) do
        $progress.print "Creating configuration backup archive: #{tar_file} ... "
        
        if Kernel.system("sh", "-c", "umask 0077; tar -cf #{tar_file} -C / etc/gitlab/gitlab.rb etc/gitlab/gitlab-secrets.json")
          $progress.puts "done".green
        else
          puts "creating archive #{tar_file} failed".red
          abort "Configuration backup failed"
        end
      end
    end
    
    def remove_old
      $progress.print "Deleting old configuration backups ... "
      keep_time = Gitlab.config.backup.keep_time.to_i
      
      if keep_time > 0
        removed = 0
        
        Dir.chdir(Gitlab.config.backup.path) do
          file_list = Dir.glob('*_gitlab_config_backup.tar')
          file_list.map! { |f| $1.to_i if f =~ /(\d+)_gitlab_config_backup.tar/ }
          
          file_list.sort.each do |timestamp|
            if Time.at(timestamp) < (Time.now - keep_time)
              if Kernel.system(*%W(rm #{timestamp}_gitlab_config_backup.tar))
                removed += 1
              end
            end
          end
        end
        
        $progress.puts "done. (#{removed} removed)".green
      else
        $progress.puts "skipping".yellow
      end
    end
    
    def configure_cron_mode
      if ENV["CRON"]
        # We need an object we can say 'puts' and 'print' to; let's use a StringIO.
        require "stringio"
        $progress = StringIO.new
      else
        $progress = $stdout
      end
    end
    
  end
end
```

Since the default wrapper (*/opt/gitlab/bin/gitlab-rake*) for invoking GitLab tasks
does not allow running tasks as root user, we have to define our own wrapper
as */root/bin/gitlab-rake-privileged*:
```
#!/bin/sh

error_echo()
{
  echo "$1" 2>& 1
}

gitlab_rails_rc='/opt/gitlab/etc/gitlab-rails/gitlab-rails-rc'
if ! [ -f ${gitlab_rails_rc} ] ; then
  error_echo "$0 error: could not load ${gitlab_rails_rc}"
  error_echo "Either you are not allowed to read the file, or it does not exist yet."
  error_echo "You can generate it with:   sudo gitlab-ctl reconfigure"
  exit 1
fi

. ${gitlab_rails_rc}

cd /opt/gitlab/embedded/service/gitlab-rails
exec /opt/gitlab/embedded/bin/chpst -e /opt/gitlab/etc/gitlab-rails/env -U ${gitlab_user} /opt/gitlab/embedded/bin/bundle exec rake "$@"
```

Make this wrapper executable:
```
chmod u=rwx,g=rx,o= /root/bin/gitlab-rake-privileged
```

Install Duply and Duplicity:
```
apt-get install duplicity duply
```

Create a basic Duply configuration file:
```
duply gitlab create
```

Enable symmetric-key encryption for the backups that will be uploaded to Amazon S3
in the newly created configuration file */root/.duply/gitlab/conf*:
```
#GPG_KEY='_KEY_ID_'
GPG_PW='...'
```

Set the Amazon S3 synchronization target, your S3 access credentials
and the source directory to be synchronized:
```
TARGET='s3://s3-eu-west-1.amazonaws.com/danielflecken.de/GitLab'
TARGET_USER='...'
TARGET_PASS='...'
SOURCE='/var/opt/gitlab/backups'
```

Make Duply create a full backup every month and delete full backups older than 12 months:
```
MAX_FULL_BACKUPS=12
MAX_FULLBKP_AGE=1M
DUPL_PARAMS="$DUPL_PARAMS --full-if-older-than $MAX_FULLBKP_AGE "
```

Create the backup script */etc/cron.daily/gitlab* which will be executed by [Cron](https://en.wikipedia.org/wiki/Cron) daily:
```
#!/bin/sh
gitlab-rake gitlab:backup:create
/root/bin/gitlab-rake-privileged gitlab:backup_config:create
duply gitlab purge-full
duply gitlab backup
```

> Background information:
> The script creates data backups (*TIMESTAMP\_gitlab\_backup.tar*)
> and configuration backups (*TIMESTAMP\_gitlab\_config\_backup.tar*) of GitLab
> in */var/opt/gitlab/backups*.
> The configuation backups contain the GitLab configuration file (*/etc/gitlab/gitlab.rb*)
> and the file holding the database encryption key used for two-factor authentication
> (*/etc/gitlab/gitlab-secrets.json*).
> The directory */var/opt/gitlab/backups* is then remotely synchronized by the script.

Make the backup script executable:
```
chmod u=rwx,g=rx,o=rx /etc/cron.daily/gitlab
```

## References

Concepts and commands for this guide are partially taken from:

- [https://about.gitlab.com/downloads/#ubuntu1404](https://about.gitlab.com/downloads/#ubuntu1404)
- [https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/doc/settings/nginx.md](https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/doc/settings/nginx.md)
- [https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/raketasks/backup_restore.md](https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/raketasks/backup_restore.md)
- [https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/doc/settings/backups.md](https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/doc/settings/backups.md)
