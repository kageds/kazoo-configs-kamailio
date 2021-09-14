## Notes on upgrading to Kamailio 5.5 and moving to Postgres backend.

Thus removing the dependency on KazooDB which is closed source.

First completely remove the current installation 

```
Removing existing kazoo-kamailo:

  yum remove kazoo-kamailio
  yum remove kamailio
  yum remove kazoo-configs-kamailio
  rm -rf /etc/kazoo
```
Next install and enable Kamailio repo (currently 5.5.0)
```
yum -y install yum-utils
yum-config-manager --add-repo https://rpm.kamailio.org/centos/kamailio.repo

# Modify kamailio.repo to enable only [kamailio-5.5.0]; all other versions should be disabled with "enabled=0"

vi /etc/yum.repos.d/kamailio.repo

[kamailio-5.5.0]
name=Kamailio - 5.5.0 - Packages for the Kamailio 5.5.0 release
baseurl=https://rpm.kamailio.org/centos/$releasever/5.5/5.5.0/$basearch/
enabled=1
metadata_expire=30d
repo_gpgcheck=0
gpgkey=https://rpm.kamailio.org/rpm-pub.key
type=rpm
skip_if_unavailable=True
```
Now install kamailio
```
yum install git kamailio kamailio-kazoo kamailio-outbound kamailio-presence kamailio-tls   kamailio-utils kamailio-uuid    kamailio-websocket kamailio-xmpp kamailio-postgresql
```
Now install latest postgres (currently 12.7)
```
# Install the repository RPM:
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-6-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo yum install -y postgresql12-server
/usr/pgsql-12/bin/postgresql-12-setup  initdb

sudo chkconfig postgresql-12 on
sudo service postgresql-12 start
```
Create Kamailio DB
```
su - postgres -c '/usr/pgsql-12/bin/createdb kamailio'
```
Add DB user and grant privileges
```
sudo su - postgres -c psql
	CREATE USER kamailio WITH PASSWORD 'kamailio';
	GRANT ALL privileges on database kamailio to kamailio;
```
Modify default postgres authentication to allow kamailio access
```
vi /var/lib/pgsql/12/data/pg_hba.conf

# "local" is for Unix domain socket connections only
local   all             all                                     password
# IPv4 local connections:
host    all             all             127.0.0.1/32            password
# IPv6 local connections:
host    all             all             ::1/128                 password
```
Increase the max number of connections and shared memory
```
vi /var/lib/pgsql/12/data/postgresql.conf

shared_buffers = 256MB
max_connections = 500 
```
Restart postgres and check kamailio access to DB
```
systemctl restart postgresql-12
psql -U kamailio -d postgres://kamailio:kamailio@127.0.0.1/kamailio
```
Get the postgres kamailio configs:
```
git clone https://github.com/kageds/kazoo-configs-kamailio /etc/kazoo
cd /etc/kazoo/kamailio
git checkout 4.3-postgres
```
Initialize the kamailio database with all the required tables:
```
psql -U kamailio -d postgres://kamailio:kamailio@127.0.0.1/kamailio -f /etc/kazoo/kamailio/db_scripts/kamailio_initdb_postgres.sql
```
Notice that the backend DB in local.cfg is now postgres
```
#!trydef KZ_DB_MODULE postgres
#!substdef "!KAMAILIO_DBMS!postgres!g"
#!substdef "!KAZOO_DB_URL!postgres://kamailio:kamailio@127.0.0.1/kamailio!g"
```
Tell kamailio where the configuration files are
```
vi /etc/sysconfig/kamailio

#
# Kamailio startup options
#

# Amount of shared memory to allocate for the running Kamailio server (in Mb)
#SHM_MEMORY=64

# Amount of per-process (package) memory to allocate for Kamailio (in Mb)
#PKG_MEMORY=4

# Enable the server to leave a core file when it crashes.
# Set this to 'yes' to enable kamailio to leave a core file when it crashes
# or 'no' to disable this feature. This option is case sensitive and only
# accepts 'yes' and 'no' and only in lowercase letters.
# On some systems (e.g. Ubuntu 6.10, Debian 4.0) it is necessary to specify
# a directory for the core files to get a dump. Look into the kamailio
# init file for an example configuration.
DUMP_CORE=no

CFGFILE=/etc/kazoo/kamailio/kamailio.cfg
```
TEMP: Patch the kazoo module
[https://github.com/kamailio/kamailio/pull/2786]
```
cp /etc/kazoo/kazoo_module/kazoo.so /usr/lib64/kamailio/modules/kazoo.so
```
Start kamailio (NOT kazoo-kamailio)
```
systemctl enable kamailio
systemctl start kamailio
```

