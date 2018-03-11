# Bacula-Client Setup:

### On Bacula-Server (addition to the prior setup):
```console
sudo mkdir /etc/bacula/conf.d
sudo nano /etc/bacula/bacula-dir.conf
```
At the end of the file add, this line:
```python
@|"find /etc/bacula/conf.d -name '*.conf' -type f -exec echo @{} \;"
```
Save and Exit.
```console
sudo nano /etc/bacula/conf.d/pools.conf
```
Add this as content:
```python
	Pool {
  			Name = RemoteFile
  			Pool Type = Backup
  			Label Format = Remote-
  			Recycle = yes                       # Bacula can automatically recycle Volumes
  			AutoPrune = yes                     # Prune expired volumes
  			Volume Retention = 365 days         # one year
    		Maximum Volume Bytes = 50G          # Limit Volume size to something reasonable
  			Maximum Volumes = 100               # Limit number of Volumes in Pool
		}
```
### Check for errors:
```console
sudo bacula-dir -tc /etc/bacula/bacula-dir.conf
```


### Bacula-Client:
```console
sudo apt-get update
```
Then install the bacula-client package:
```console
sudo apt-get install bacula-client
```
Configure Client:
```console
nano /etc/hostname 
```
```python
	bacula-client               # for example
```
```console
nano /etc/hosts
```
```python
	10.1.12.250 bacula-client.openstacklocal bacula-client             # if they don’t already exists
```
```console
nano /etc/bacula/bacula-fd.conf
```
Edit this Name part to backup-server-hostname and remember the password (ca line 13):
```python
	Director {
  		Name = backup-sam-dir
  		Password = "HWddrD5uk72XVhZhN7vJgLFvqmw4SnIFb"
		}

ca line 31:
	FileDaemon {                          # this is me
  		Name = logger-kibana-fd
  		FDport = 9102                  # where we listen for the director
  		WorkingDirectory = /var/lib/bacula
  		Pid Directory = /var/run/bacula
  		Maximum Concurrent Jobs = 20
 		FDAddress = logger-kibana.openstacklocal
	}

ca line 41:
	Messages {
  		Name = Standard
  		director = backup-sam-dir = all, !skipped, !restored
	}
```
### Check for errors:
```console
sudo bacula-fd -tc /etc/bacula/bacula-fd.conf
```
If echo $? == 0
```console
sudo service bacula-fd restart
```
Let's set up a directory that the Bacula Server can restore files to. Create the file structure and lock down the permissions and ownership for security with the following commands:
```console
sudo mkdir -p /bacula/restore
sudo chown -R bacula:bacula /bacula
sudo chmod -R 700 /bacula
```
### On Bacula-Server:
```console
sudo nano /etc/bacula/conf.d/filesets.conf
```
* The FileSet Name must be unique
* Include any files or partitions that you want to have backups of
* Exclude any files that you don't want to back up, but were selected as a result of existing within an included file
```python	
	FileSet {
  		Name = "Home and Etc"
  		Include {
    			Options {
      				signature = MD5
      				compression = GZIP
    			}
    			File = /home
    			File = /etc
    			File = /root
  		}
  		Exclude {
    		File = /home/bacula/
  		}
	}

```
Add Client and Backup Job to Bacula Server:
```console
sudo nano /etc/bacula/conf.d/clients.conf
```
Add Client Resource:
```python
	Client {
  		Name = logger-kibana-fd
  		Address = logger-kibana.openstacklocal
  		FDPort = 9102
  		Catalog = MyCatalog
  		Password = "HWddrD5uk72XVhZhN7vJgLFvqmw4SnIFb"          # password for Remote FileDaemon
  		File Retention = 30 days            # 30 days
  		Job Retention = 6 months            # six months
  		AutoPrune = yes                     # Prune expired Jobs/Files
	}
```
Create a backup job:
```python
	Job {
  		Name = "Backuplogger-kibana"
  		JobDefs = "DefaultJob"
  		Client = logger-kibana-fd
  		Pool = RemoteFile
  		FileSet="Home and Etc"
	}
```
Verify Director Configuration:
```console
sudo bacula-dir -tc /etc/bacula/bacula-dir.conf
```
Restart Bacula Director
```console
sudo service bacula-director restart
```
Test Backup Job:
```console
	run -> 4 -> yes
```
Perform Restore:
```console
    restore all -> 5 -> 2 (or depending on the which client to restore to) -> done -> yes
```
