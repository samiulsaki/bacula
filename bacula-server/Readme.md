# Bacula-Server Setup:


### Bacula-Server:
```console
apt-get update
apt-get install mysql-server
``` 
Enter a Password (remember the password)
```console
apt-get install bacula-server bacula-client
```

* Select YES for configuring database
* Enter mysql password (from previous step)
* Enter a new password and confirm

```console
chmod 755 /etc/bacula/scripts/delete_catalog_backup
```
### Mount the volume:
```console
mkfs.ext4 /dev/vdb
mount /dev/vdb /mnt
mkdir -p /mnt/bacula/backup /mnt/bacula/restore
chown -R bacula:bacula /mnt/bacula
chmod -R 700 /mnt/bacula
```

### Configure Bacula Director:
```console
nano /etc/bacula/bacula-dir.conf
```
Find the Job resource with a name of "BackupClient1” (ca line 45) and rename the Name part:
```python
		Job {
  		Name = "BackupLocalFiles"
  		JobDefs = "DefaultJob"
		}
```
Find the Job resource that is named "RestoreFiles" (ca line 77) and rename the Name and Where part:
```python
	Job {
  		Name = "RestoreLocalFiles"
  		Type = Restore
 		Client=backup-sam-fd
  		FileSet="Full Set"
		Storage = File
  		Pool = Default
  		Messages = Standard
  		Where = /mnt/bacula/restore
	}
```
### Configure File Set: (ca line 90)
```python
	FileSet {
  		Name = "Full Set"
  		Include {
    		Options {
      		signature = MD5
      		compression = GZIP
    		}
	    File = /
  	}
  	Exclude {
    		File = /var/lib/bacula
    		File = /mnt/bacula
		File = /proc
    		File = /tmp
    		File = /.journal
    		File = /.fsck
  	}
}
```
### Configure Storage Daemon Connection: 
Use the backup server FQDN here (ca line 185):
```python
	Storage {
 		Name = File
		# Do not use "localhost" here
  		Address = backup-sam.openstacklocal                # N.B. Use a fully qualified name here
  		SDPort = 9103
  		Password = "tL3-S6OfmJhkT6CGN9axDWm-64uc18oko"
  		Device = FileStorage
  		Media Type = File
	}
```
### Configure Pool:
Add a line that specifies a Label Format (ca line 282):
```python
    Pool {
  		Name = File
  		Pool Type = Backup
  		Label Format = Local-
  		Recycle = yes                       # Bacula can automatically recycle Volumes
  		AutoPrune = yes                     # Prune expired volumes
  		Volume Retention = 365 days         # one year
  		Maximum Volume Bytes = 50G          # Limit Volume size to something reasonable
  		Maximum Volumes = 100               # Limit number of Volumes in Pool
	}
```
### Check Director Configuration:
```console
bacula-dir -tc /etc/bacula/bacula-dir.conf
```
### Configure Storage Daemon:
```console
nano /etc/bacula/bacula-sd.conf
```
(ca line 13):
```python	
	Storage {                             # definition of myself
  		Name = backup-sam-sd
  		SDPort = 9103                  # Director's port
  		WorkingDirectory = "/var/lib/bacula"
  		Pid Directory = "/var/run/bacula"
  		Maximum Concurrent Jobs = 20
  		SDAddress = backup-sam.openstacklocal
	}
```
(ca line 53):
```python
	Device {
  		Name = FileStorage
  		Media Type = File
  		Archive Device = /mnt/bacula/backup
  		LabelMedia = yes;                   # lets Bacula label unlabeled media
  		Random Access = Yes;
  		AutomaticMount = yes;               # when device opened, read it
  		RemovableMedia = no;
  		AlwaysOpen = no;
	}
```
### Verify Storage Daemon Configuration:
```console
bacula-sd -tc /etc/bacula/bacula-sd.conf
```
### Restart Bacula Director and Storage Daemon:
```console
sudo service bacula-director restart
sudo service bacula-sd restart
```
### To backup:
```console
run -> 1 -> yes
```
### To restore:
```console
restore all -> 5 -> 2
```

### Delete Restored Files
You may want to delete the restored files to free up disk space. To do so, use this command:
```console
sudo -u root bash -c "rm -rf /bacula/restore/*"
```
