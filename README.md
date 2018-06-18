# WMAU Infrastructure
Wikimedia Australia infrastructure management scripts.

- https://wikimedia.org.au
- Author: Michael Billington, Wikimedia Australia Inc.

The following infrastructure is managed by these scripts:

- A webserver containing the public website
 - Single node MediaWiki family
 - CiviCRM installed on Drupal
 - WordPress blog

## Setup checklist
Requirements:

- Access to Debian 8 virtual machines to administer
  - SSH public key auth is set up so that you can connect to the machine as a regular user
  - Your account has access to 'sudo' w/ password.
  - NOTE: The server will be configured withSSH options 'PermitRootLogin no', 'PasswordAuthentication no'.
- Access to files not included in this repository:
  - A WMAU backup archive (store under backup/).
  - Vars for credentials to use (store under group_vars/).

Create the inventories called `production` and `staging`. Production should include the current live machines, while staging should contain local machines for you to use for testing changes on a copy of the infrastructure.

Example `production` inventory:

	[all:vars]
	wmau_domain=wikimedia.org.au
  tls_selfsign=False

	[web]
	osiris.wikimedia.org.au ansible_connection=ssh set_remote_user=no

	[backup]
	server2.wikimedia.org.au ansible_connection=ssh

Example `staging` inventory:

	[all:vars]
	wmau_domain=my-vm
  tls_selfsign=True

	[web]
	my-vm ansible_connection=ssh

	[backup]
	my-other-vm ansible_connection=ssh

Edit `group_vars/web` to include correct credentials and paths.

Deploy to your staging inventory:

    ansible-playbook -i staging site.yml --ask-become-pass

### First local deployment

The first deployment to a machine must include the destructive database and image asset restore operations.

To run this on your staging inventory, run:

    ansible-playbook -i staging site.yml --ask-become-pass --extra-vars="restore=yes"

### Applying changes

For new extensions, software updates, or configuration changes, edit this playbook, and re-run it on your a staging environment:

    ansible-playbook -i staging site.yml --ask-become-pass

Once you have tested the changes, commit them to git, and deploy them to the production environment:

    ansible-playbook -i production site.yml --ask-become-pass

## Maintentance tasks

### Backups

Database dumps will be created daily on the server, and file uploads will appear in the live web directory.

To replace your local 'backups' directory with copies of this server-side data, use `backup=yes`. To skip applying updates (faster/safer), also filter down to backup tasks only:

    ansible-playbook -i production site.yml --ask-become-pass --extra-vars="backup=yes" --tags "backup"

### Updates

Regularly run the playbook over the staging and production environments to pull in updates and patches to MediaWiki. Generally, the longer you run without an update, the more difficult the update will be!

    ansible-playbook -i staging site.yml --ask-become-pass
    ansible-playbook -i production site.yml --ask-become-pass

The playbook only tracks major releases of MediaWiki, as there is generally no need to be able to re-create older deployments.

### Manual tasks not yet performed in Ansible

- Backups
- Updates to Wordpress, Drupal
- Monitoring of server

## Known deficiencies
- SELinux not configured (security)
- Due to Ansible/rsync integration issue, the need for a user password when using sudo is temporarily disabled during a 'restore=yes' and 'restore=backup' run.

### Future improvements
- Automation of manual tasks above
- Caching setup
- HHVM install
- Install Visual Editor and other extensions (same as WMF wikis)

## Additional notes
To install on a new 'Linode' Debian 8.1 machine, follow these steps.

SSH in as root and add a new account.

	# adduser example-user
	Adding user `example-user' ...
	Adding new group `example-user' (1000) ...
	Adding new user `example-user' (1000) with group `example-user' ...
	Creating home directory `/home/example-user' ...
	Copying files from `/etc/skel' ...
	Enter new UNIX password:
	Retype new UNIX password:
	passwd: password updated successfully
	Changing the user information for example-user
	Enter the new value, or press ENTER for the default
		Full Name []: Example User
		Room Number []:
		Work Phone []:
		Home Phone []:
		Other []:
	Is the information correct? [Y/n] Y
	root@localhost:~# usermod -a -G sudo example-user

And then from a remote machine, copy your ssh credentials:

    ssh-copy-id example-user@ip.address.here.xyz

Configure the machine to be accessible via its correct DNS name, wait 15 minutes
for DNS changes to be visible, and then run the initial deployment as above.
