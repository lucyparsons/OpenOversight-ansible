OpenOversight-ansible
=========

Ansible playbook for deploying multi-environment dockerized OpenOversight. By default, this playbook also installs Traefik alongside OpenOversight to act as a reverse proxy.

Requirements
------------

This playbook is designed to be deployed to Debian or Ubuntu Linux. The server must have Python 3 and sudo installed, and your user account should be in the `sudo` group so it can escalate privileges when needed.

Setup
-----

You'll first need to create an inventory file `hosts` that includes one or more IP addresses or fully qualified domain names of your server(s), one per line:

```
ec2-100-25-161-239.compute-1.amazonaws.com
ec2-3-216-167-202.compute-1.amazonaws.com
```

Next, create a `host_vars` directory and within that a file for each server, e.g. `openoversight.com.yml`. This YAML file will contain all the variables to customize the deployment.

You can deploy multiple instances of OpenOversight (e.g. staging and production) to one server, or each instance can be on its own server. If you want to deploy multiple instances on one server, you'll have multiple entries under the `openoversight_environments` variable in your YAML file (see below). If you instead have different servers for different instances, you will have multiple YAML files, one for each server, each with only one entry in `openoversight_environments`.

Configuration Variables
-----------------------

Below is a sample configuration for multiple instances deployed to one server. Customize to fit your deployment. Note that the `openoversight_password` variable is in the format of `/etc/shadow`.

```YAML
---
openoversight_user: ubuntu
openoversight_user_home_dir: /home/ubuntu
openoversight_password: $5$k90iYj3.mL3HYivL$gGoFuYkChipnjXTHC.IsXhcJR2LR4K9HYv3NkCIlgKC
openoversight_git_repo: https://github.com/lucyparsons/OpenOversight.git
openoversight_db_name: openoversight-prod
openoversight_db_user: openoversight
openoversight_db_password: terriblepassword
openoversight_postgres_version: 12
openoversight_secret_key: 4Q6ZaQQdiqtmvZaxP1If
openoversight_mail_server: smtp.gmail.com
openoversight_mail_username: mail_username
openoversight_mail_password: mail_password
openoversight_environments:
  - name: production
    path: /usr/src/openoversight/production
    domain: openoversight.com
    git_version: main
    s3_bucket: openoversight-photos-prod
    approve_registrations: false
    disable_registration: false
    internal_port: 3000
    sync_photos: yes
    traefik_enabled: yes
    backup_dir: /var/backups/openoversight
  - name: staging
    path: /usr/src/openoversight/staging
    domain: staging.openoversight.com
    git_version: develop
    s3_bucket: openoversight-photos-staging
    approve_registrations: false
    disable_registration: false
    internal_port: 3001
    sync_photos: no
    traefik_enabled: yes
aws_secret_access_key: AS3HMDTB8TS440V9TD7T
aws_access_key_id: AZwA1IAlYlAo4ugm2R8ykZpMJX8LN4p6zwJmUm2j
aws_default_region: us-east-1
traefik_user: ubuntu
traefik_dir: /etc/traefik
traefik_email: postmaster@openoversight.com
```

Additional optional variables are below:
```YAML
recaptcha_v3_site_key: XQngwEWF52sxVW0Dwzai1LVnxvOkHbUy3OMp7efw
recaptcha_v3_secret_key: wNsR9aQjHTaLTEHpmXdX7VwSB9xdcso7H6zwT2D7
recaptcha_v3_threshold: 0.5
openoversight_mail_subject_prefix: "[OpenOversight]"
openoversight_mail_sender: "OpenOversight <OpenOversight@gmail.com>"
```

Running the Playbook
--------------------

For convenience, you can first run the playbook using the `docker` tag to install Docker if needed:

```
ansible-playbook -K -i hosts --tags docker --private-key=~/.ssh/openoversight.pem -u ubuntu playbook.yml
```

Here `-K` prompts you for sudo password, `hosts` is the inventory file, `--private-key` specifies the SSH private key to use, and `-u ubuntu` says to run it as the ubuntu user. 

Then to deploy OpenOversight, just remove the `--tags docker`:

```
ansible-playbook -K -i hosts --private-key=~/.ssh/openoversight.pem -u ubuntu playbook.yml
```

If you want to deploy OpenOversight *without* Traefik:

```
ansible-playbook -K -i hosts --skip-tags traefik --private-key=~/.ssh/openoversight.pem -u ubuntu playbook.yml
```

Post-deployment
---------------

For a new deployment, you'll likely want to import an existing OpenOversight database backup:

```
cat backup-production-20200710-131815.sql | docker exec -i openoversight-db-production psql postgresql://openoversight:terriblepassword@localhost -f -
```

A new deployment will also need to build web assets using Yarn:

```
docker exec -it openoversight-production yarn build
```

Backup and Photo Sync
---------------------

A backup script is installed that locally backs up the database on a weekly basis. You can optionally send these database backups to S3 for remote storage by including the `backup_s3_bucket` variable in one of your `openoversight_environments`.

You can also optionally sync the site's S3 photos to a local directory on your server with the `photo_sync.sh` script. By setting the `sync_photos: yes` variable in your `openoversight_environments`, as in the example above, the photo sync script will be installed and run weekly.

License
-------

GPLv3
