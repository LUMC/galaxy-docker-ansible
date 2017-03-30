# galaxy-docker-ansible

This project contains the roles needed to install bjgruening/galaxy-docker-stable image on an 
ubuntu server.

## Getting started
1. Clone the repository to your local computer.
2. [Set up ansible](http://docs.ansible.com/ansible/intro_installation.html)
  * Ansible version 2.2 is tested and required.
3. Set up a passwordless sudo user on the remote host and set up an ssh key pair.
4. Make a new hosts file by copying `hosts.sample` to `hosts` and setup your galaxy host.
5. Create a new configuration directory by copying `host_vars/example_host` to `host_vars/HOSTNAME`. Hostname should be equal to that specified in `hosts`
6. Create a new files directory by copying `files/example_host` to `files/HOSTNAME`
 
## Configuring your installation.
Settings files are located in `host_vars/HOSTNAME`. `docker.settings`, `galaxy.settings` and `port.settings` should be checked and if necessary changed.
Also tool lists can be added to install a set of tools.

### docker.settings
Variable | Function
---|---
docker_default_location | Where docker stores the images and containers. Use a volume with ample disk space.
docker_image | The docker image. Defaults to "bgruening/galaxy-stable:latest" but it's better to tag it with a version number. (i.e. 17.01)
docker_user | a user that will be created without sudo rights on the remote machine.
docker_container_name | What name the container gets for easy access using docker commands. Default is "galaxy".

### galaxy.settings
Variable | Function
---|---
galaxy_admin_user | e-mail address of the admin user. This variable is obligatory
galaxy_master_api_key | The master api key. Always set this value to something unique.
galaxy_brand | The galaxy brand name
optional_environment_settings | This is a YAML dictionary that takes any docker environment values. See the documentation of [bjgruening/galaxy-stable](https://github.com/bgruening/docker-galaxy-stable/blob/master/README.md) which options are available.

### port.settings
Variable | Function
---|---
galaxy_web_port | default 80
galaxy_ftp_port | default 8021
galaxy_sftp_port | default 8022
ufw_profile | Firewall access is managed by a ufw profile to prevent the firewall to clog up with orphaned rules. Default ufw_profile name is "galaxy"

### Tools
Tool lists can be added to `files/HOSTNAME/tools`. 
An example tool list can be found in `files/example_host/tools`.
If no files are present, this step will be automatically skipped.

## Starting a galaxy instance on a remote machine.

To install docker, fetch the image, run it and install all the tools run:

```bash
ansible-playbook main.yml -e "hosts=HOSTNAME run=install"   
```

If you did not set up passwordless sudo you can add the -K parameter and type in the sudo password.


Alternatively you can set up your machine step by step.

### Install docker on the remote machine
```bash
ansible-playbook main.yml -e "hosts=HOSTNAME run=installdocker"   
```

### Run the galaxy docker image
```bash 
ansible-playbook main.yml -e "hosts=HOSTNAME run=rundockergalaxy"   
```

### Install tool lists on the galaxy instance
```bash
ansible-playbook main.yml -e "hosts=HOSTNAME run=installtools"   
```

oThe tool lists need to be in the directory specified in tool_list_dir in galaxydocker.config.
Example tool lists can be found at [the galaxy project's github](https://github.com/galaxyproject/ansible-galaxy-tools/blob/master/files/tool_list.yaml.sample)

## Testing a new version of the image.

If bjgruening updates the docker image to a newer version than this can be tested as follows:
1. Open the `host_vars/HOSTNAME/upgrade.settings` file
2. Set the settings for the test instance in the test_upgrade dictionary. Make sure the port mappings don't overlap with the running instance. Additional settings can be added to the dictionary.
3. Run `ansible-playbook main.yml -e "hosts=HOSTNAME run=testupgrade"`
4. Check if the galaxy instance is running properly and if history is kept.
(Tools won't run and data will not be included)
5. Settings are stored in `/export/galaxy-central/config`, any new config files are automatically copied to this directory if these do not yet exist.
Existing files are not replaced. To check for any new features you can diff `/export/.distribution_config` and `/export/galaxy-central/config`

## Upgrade the running instance to a new image
1. Make sure there are no jobs running on your instance. As an admin you can hold all new jobs so they will wait until the image is upgraded.
2. Update the version tag of docker_image in `host_vars\HOSTNAME\docker.settings`
3. run `ansible-playbook main.yml -e "hosts=HOSTNAME run=upgrade"`

There is a setting overwrite_config_files in migrate.settings. Default is False. 
If set to True this will overwrite all your config files with the .distribution_config files.

## Backing up your export folder
For backing up your export folder use `ansible-playbook main.yml -e "hosts=HOSTNAME run=backupgalaxy`
This role is not very extensive and may need extension based upon your needs.

Settings are in the `host_vars/HOSTNAME/backup.settings` file

Variable | Function
---|---
method | Can be set to rsync or archive. Rsync is useful for hourly or daily backups. Archive stores an archive but does not allow incremental backups
backup_location | Absolute path to the location. If a remote location is chosen use the following syntax user@host::dest. Remote locations can only be chosen when the method is rsync
prefix, backup_name, postfix | set the filename
remote_backup | set to True if a remote location is chosen
rsync_compression_level | Compression level to limit the bandwith used when a backup is made on a remote location
compression_format | Can be 'gz,' 'zip' or 'bz2'. Only when method=archive

The archive method is only functional as of ansible 2.3. Therefore the archive function is still commented out in the tasks/main.yml file.
It can be enabled when ansible 2.3 is released as stable.

## Removing the galaxy docker instance.
`ansible-playbook main.yml -e "hosts=HOSTNAME run=deletegalaxy` does the following things:
- deletes the container
- removes the firewall exception and removes the profile

If `delete_files=True` is added then it will also delete the export folder 


