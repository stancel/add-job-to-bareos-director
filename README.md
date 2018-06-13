add-job-to-bareos-director
=========

This role is to be used by those that use [Bareos Backup System](https://www.bareos.org/en/) which is an enterprise grade open source backup management system that comes with a useful [Web UI](https://www.bareos.org/en/bareos-webui.html), making it very easy to restore any backup. The slowest part of dealing with Bareos is adding new machines and/or files into the system. 

This role allows for a fast and easy way to add new machines or items to be systematically backed up. The ansible host to run with this role should be your Bareos Server (where bareos-dir and bareos-sd files are stored).

Bareos is a derivation of [Bacula](https://blog.bacula.org/what-is-bacula/). This Ansible role may work with Bacula, but has only been tested with Bareos.

Bareos does have the ability to backup laptops (Mac, Linux and Windows) and other devices that do not have static IP addresses assigned to them, but this role does not cover those cases at this time. This role assumes that the bareos director (bareos-dir) component of the system will be able to reach out to the `server_fqdn` at the time of the backup and coordinate with the bareos file daemon (bareos-fd) to retrieve the needed files and pass them off to the bareos storage daemon (bareos-sd) for permanent storage.

Requirements
------------

You need to use and have a fully setup [Bareos Backup System](https://www.bareos.org/en/) on version 17.x or higher. I started using Bareos before Ansible and did not use Ansible to setup my instances, but did find [this Ansible role](https://github.com/bashrc666/ansible-role-bareos) recently for installing the Bareos system.

Role Variables
--------------

The hostname of the machine you are backing up

	`host_name: "my-important-crm-server"`
	
The fully qualified domain name or IP address of the machine you are adding to be backed up

	`server_fqdn: "crm.example.com"`
	
This is a very important one. It needs to match the client_password entered when adding the bareos client program to the machine you want to backup. You will probably want to see the [Ansible role I wrote for that purpose (install-bareos-client) here](https://github.com/stancel/install-bareos-client). The value for this is usually a very long string (47 characters) of random characters like below.

	`client_password: "kAgt2SJTysbg5iRpcnj5XRexuEnDieGCetCXNQrYGuzNxCf"`

The storage director password variable is the bareos-dir password found on your Bareos Server. It is a good idea to use an environment variable (like below) or Ansible Vault in order to store this value and retrieve it into the `storage_director_password` variable. 

	`storage_director_password: "{{ lookup('env', 'BAREOS_STORAGE_PASSWORD') }}"`


Backup Level - Full, Incremental, Differential
* Incremental - all ﬁles changed since the last Full, Diﬀerential, or Incremental backup started
* Diﬀerential - all ﬁles speciﬁed in the FileSet that have changed since the last successful *Full* backup will be backed up
* Full - all ﬁles in the FileSet whether or not they have changed will be backed up

	`level: Incremental`

Permits you to control the order in which your jobs will be run by specifying a positive non-zero number. The higher the number, the lower the job priority. 

	`priority: 6`

The backup schedule you want to run for this job/machine that are adding with this role. [Official Documentation Here](http://doc.bareos.org/master/html/bareos-manual-main-reference.html#x1-1380009.4)

```
	backup_schedule:
	  - Run = Full sun at 02:00
	  - Run = Incremental mon-sat at 02:00
```

As you'll see in the example below I use this like a file path, but you have some other interesting options as well. I recommend checking them out in the [Bareos Documentation here](http://doc.bareos.org/master/html/bareos-manual-main-reference.html#directiveSdDeviceArchive%20Device). You can also backup to distributed file systems like GlusterFS and object storage (AWS S3), tape, etc.

	`archive_device: "/z-storage/backups/{{ host_name }}"`


A list of directories and/or ﬁles to be processed in the backup job. [Official Documentation Here](http://doc.bareos.org/master/html/bareos-manual-main-reference.html#x1-1410009.5.1)

```
	filesets_to_backup:
	  - File = "/var/www/html"
	  - File = "/path/to/your/directory"
	  - File = "/backups/mysql/current"
```  


A list of directories and/or ﬁles to be excluded in the backup job. [Official Documentation Here](http://doc.bareos.org/master/html/bareos-manual-main-reference.html#x1-1430009.5.2). Don't actually store your git files/directory under a webserver's DocumentRoot. This is just meant to demonstrate the use case of excluding files or directories from a directory that you are backing up.

```
	filesets_to_exclude:
	  - File = "/var/www/html/.github"
	  - File = "/var/www/html/.git"
	  - File = "/var/www/html/.gitignore"
```

There are some additional straightforward variables whose values are set in the [defaults/main.yml file](../blob/master/defaults/main.yml). They are:

	`backup_name: "{{ host_name }} Backup"`

Individual File Records Storage Length - affects browsing files in GUI

	`file_retention: 90 days`
	
Job Records in the DB - how long to store info/metadata about these backups

	`job_retention: 6 months`
	
How long to actually store the backup files

	`volume_retention_full: 12 months`
	
Maximum size of each backup volume/file

	`maximum_volume_bytes_full: 10G`
	
Limit number of Volumes in Pool

	`maximum_volumes_full: 10`

How long to actually store the backup files

	`volume_retention_incremental: 12 months`

Maximum size of each backup volume/file

	`maximum_volume_bytes_incremental: 10G`

Limit number of Volumes in Pool

	`maximum_volumes_incremental: 10`



Dependencies
------------

No dependencies on other Ansible roles.

Example Playbook
----------------


	- hosts: your_bareos_server
	  vars_files:
	    - vars/main.yml
	  roles:
	    - { role: stancel.add-job-to-bareos-director }


or 


	- hosts: your_bareos_server
	  vars:
		host_name: "my-important-crm-server"
		server_fqdn: "crm.example.com"
		client_password: "kAgt2SJTysbg5iRpcnj5XRexuEnDieGCetCXNQrYGuzNxCf"
		storage_director_password: "{{ lookup('env', 'BAREOS_STORAGE_PASSWORD') }}" 
		level: Incremental
		priority: 6
		backup_schedule:
		- Run = Full sun at 02:00
		- Run = Incremental mon-sat at 02:00
		archive_device: "/z-storage/backups/{{ host_name }}"
		filesets_to_backup:
		- File = "/var/www/html"
		- File = "/path/to/your/directory"
		- File = "/backups/mysql/current"
		filesets_to_exclude:
		- File = "/var/www/html/.github"
		- File = "/var/www/html/.git"
		- File = "/var/www/html/.gitignore"
	  roles:
	    - { role: stancel.add-job-to-bareos-director }


License
-------

GPLv3

Author Information
------------------

Brad Stancel