***MySql 5.6 error (Table 'mysql.user' doesn't exist) when started as a daemon***








**First check /etc/init.d/mysql File see if every thing is right there as according to your mysql base path and binary path**


```
[root@puppet mysql]# cat /etc/init.d/mysql
#!/bin/sh
# Copyright Abandoned 1996 TCX DataKonsult AB & Monty Program KB & Detron HB
# This file is public domain and comes with NO WARRANTY of any kind

# MySQL daemon start/stop script.

# Usually this is put in /etc/init.d (at least on machines SYSV R4 based
# systems) and linked to /etc/rc3.d/S99mysql and /etc/rc0.d/K01mysql.
# When this is done the mysql server will be started when the machine is
# started and shut down when the systems goes down.

# Comments to support chkconfig on RedHat Linux
# chkconfig: 2345 64 36
# description: A very fast and reliable SQL database engine.

# Comments to support LSB init script conventions
### BEGIN INIT INFO
# Provides: mysql
# Required-Start: $local_fs $network $remote_fs
# Should-Start: ypbind nscd ldap ntpd xntpd
# Required-Stop: $local_fs $network $remote_fs
# Default-Start:  2 3 4 5
# Default-Stop: 0 1 6
# Short-Description: start and stop MySQL
# Description: MySQL is a very fast and reliable SQL database engine.
### END INIT INFO

# If you install MySQL on some other places than /usr, then you
# have to do one of the following things for this script to work:
#
# - Run this script from within the MySQL installation directory
# - Create a /etc/my.cnf file with the following information:
#   [mysqld]
#   basedir=<path-to-mysql-installation-directory>
# - Add the above to any other configuration file (for example ~/.my.ini)
#   and copy my_print_defaults to /usr/bin
# - Add the path to the mysql-installation-directory to the basedir variable
#   below.
#
# If you want to affect other MySQL variables, you should make your changes
# in the /etc/my.cnf, ~/.my.cnf or other MySQL configuration files.

# If you change base dir, you must also change datadir. These may get
# overwritten by settings in the MySQL configuration files.

basedir=
datadir=

# Default value, in seconds, afterwhich the script should timeout waiting
# for server start.
# Value here is overriden by value in my.cnf.
# 0 means don't wait at all
# Negative numbers mean to wait indefinitely
service_startup_timeout=900

# Lock directory for RedHat / SuSE.
lockdir='/var/lock/subsys'
lock_file_path="$lockdir/mysql"

# The following variables are only set for letting mysql.server find things.

# Set some defaults
mysqld_pid_file_path=
if test -z "$basedir"
then
  basedir=/usr
  bindir=/usr/bin
  if test -z "$datadir"
  then
    datadir=/var/lib/mysql
  fi
  sbindir=/usr/sbin
  libexecdir=/usr/sbin
else
  bindir="$basedir/bin"
  if test -z "$datadir"
  then
    datadir="$basedir/data"
  fi
  sbindir="$basedir/sbin"
  libexecdir="$basedir/libexec"
fi

# datadir_set is used to determine if datadir was set (and so should be
# *not* set inside of the --basedir= handler.)
datadir_set=

#
# Use LSB init script functions for printing messages, if possible
#
lsb_functions="/lib/lsb/init-functions"
if test -f $lsb_functions ; then
  . $lsb_functions
else
  log_success_msg()
  {
    echo " SUCCESS! $@"
  }
  log_failure_msg()
  {
    echo " ERROR! $@"
  }
fi

PATH="/sbin:/usr/sbin:/bin:/usr/bin:$basedir/bin"
export PATH

mode=$1    # start or stop

[ $# -ge 1 ] && shift


other_args="$*"   # uncommon, but needed when called from an RPM upgrade action
           # Expected: "--skip-networking --skip-grant-tables"
           # They are not checked here, intentionally, as it is the resposibility
           # of the "spec" file author to give correct arguments only.

case `echo "testing\c"`,`echo -n testing` in
    *c*,-n*) echo_n=   echo_c=     ;;
    *c*,*)   echo_n=-n echo_c=     ;;
    *)       echo_n=   echo_c='\c' ;;
esac

parse_server_arguments() {
  for arg do
    case "$arg" in
      --basedir=*)  basedir=`echo "$arg" | sed -e 's/^[^=]*=//'`
                    bindir="$basedir/bin"
                    if test -z "$datadir_set"; then
                      datadir="$basedir/data"
                    fi
                    sbindir="$basedir/sbin"
                    libexecdir="$basedir/libexec"
        ;;
      --datadir=*)  datadir=`echo "$arg" | sed -e 's/^[^=]*=//'`
                    datadir_set=1
        ;;
      --pid-file=*) mysqld_pid_file_path=`echo "$arg" | sed -e 's/^[^=]*=//'` ;;
      --service-startup-timeout=*) service_startup_timeout=`echo "$arg" | sed -e 's/^[^=]*=//'` ;;
    esac
  done
}

wait_for_pid () {
  verb="$1"           # created | removed
  pid="$2"            # process ID of the program operating on the pid-file
  pid_file_path="$3" # path to the PID file.

  i=0
  avoid_race_condition="by checking again"

  while test $i -ne $service_startup_timeout ; do

    case "$verb" in
      'created')
        # wait for a PID-file to pop into existence.
        test -s "$pid_file_path" && i='' && break
        ;;
      'removed')
        # wait for this PID-file to disappear
        test ! -s "$pid_file_path" && i='' && break
        ;;
      *)
        echo "wait_for_pid () usage: wait_for_pid created|removed pid pid_file_path"
        exit 1
        ;;
    esac

    # if server isn't running, then pid-file will never be updated
    if test -n "$pid"; then
      if kill -0 "$pid" 2>/dev/null; then
        :  # the server still runs
      else
        # The server may have exited between the last pid-file check and now.
        if test -n "$avoid_race_condition"; then
          avoid_race_condition=""
          continue  # Check again.
        fi

        # there's nothing that will affect the file.
        log_failure_msg "The server quit without updating PID file ($pid_file_path)."
        return 1  # not waiting any more.
      fi
    fi

    echo $echo_n ".$echo_c"
    i=`expr $i + 1`
    sleep 1

  done

  if test -z "$i" ; then
    log_success_msg
    return 0
  else
    log_failure_msg
    return 1
  fi
}

# Get arguments from the my.cnf file,
# the only group, which is read from now on is [mysqld]
if test -x "$bindir/my_print_defaults";  then
  print_defaults="$bindir/my_print_defaults"
else
  # Try to find basedir in /etc/my.cnf
  conf=/etc/my.cnf
  print_defaults=
  if test -r $conf
  then
    subpat='^[^=]*basedir[^=]*=\(.*\)$'
    dirs=`sed -e "/$subpat/!d" -e 's//\1/' $conf`
    for d in $dirs
    do
      d=`echo $d | sed -e 's/[  ]//g'`
      if test -x "$d/bin/my_print_defaults"
      then
        print_defaults="$d/bin/my_print_defaults"
        break
      fi
    done
  fi

  # Hope it's in the PATH ... but I doubt it
  test -z "$print_defaults" && print_defaults="my_print_defaults"
fi

#
# Read defaults file from 'basedir'.   If there is no defaults file there
# check if it's in the old (depricated) place (datadir) and read it from there
#

extra_args=""
if test -r "$basedir/my.cnf"
then
  extra_args="-e $basedir/my.cnf"
else
  if test -r "$datadir/my.cnf"
  then
    extra_args="-e $datadir/my.cnf"
  fi
fi

parse_server_arguments `$print_defaults $extra_args mysqld server mysql_server mysql.server`

#
# Set pid file if not given
#
if test -z "$mysqld_pid_file_path"
then
  mysqld_pid_file_path=$datadir/`hostname`.pid
else
  case "$mysqld_pid_file_path" in
    /* ) ;;
    * )  mysqld_pid_file_path="$datadir/$mysqld_pid_file_path" ;;
  esac
fi

case "$mode" in
  'start')
    # Start daemon

    # Safeguard (relative paths, core dumps..)
    cd $basedir

    echo $echo_n "Starting MySQL"
    if test -x $bindir/mysqld_safe
    then
      # Give extra arguments to mysqld with the my.cnf file. This script
      # may be overwritten at next upgrade.
      $bindir/mysqld_safe --datadir="$datadir" --pid-file="$mysqld_pid_file_path" $other_args >/dev/null &
      wait_for_pid created "$!" "$mysqld_pid_file_path"; return_value=$?

      # Make lock for RedHat / SuSE
      if test -w "$lockdir"
      then
        touch "$lock_file_path"
      fi

      exit $return_value
    else
      log_failure_msg "Couldn't find MySQL server ($bindir/mysqld_safe)"
    fi
    ;;

  'stop')
    # Stop daemon. We use a signal here to avoid having to know the
    # root password.

    if test -s "$mysqld_pid_file_path"
    then
      mysqld_pid=`cat "$mysqld_pid_file_path"`

      if (kill -0 $mysqld_pid 2>/dev/null)
      then
        echo $echo_n "Shutting down MySQL"
        kill $mysqld_pid
        # mysqld should remove the pid file when it exits, so wait for it.
        wait_for_pid removed "$mysqld_pid" "$mysqld_pid_file_path"; return_value=$?
      else
        log_failure_msg "MySQL server process #$mysqld_pid is not running!"
        rm "$mysqld_pid_file_path"
      fi

      # Delete lock for RedHat / SuSE
      if test -f "$lock_file_path"
      then
        rm -f "$lock_file_path"
      fi
      exit $return_value
    else
      log_failure_msg "MySQL server PID file could not be found!"
    fi
    ;;

  'restart')
    # Stop the service and regardless of whether it was
    # running or not, start it again.
    if $0 stop  $other_args; then
      $0 start $other_args
    else
      log_failure_msg "Failed to stop running server, so refusing to try to start."
      exit 1
    fi
    ;;

  'reload'|'force-reload')
    if test -s "$mysqld_pid_file_path" ; then
      read mysqld_pid <  "$mysqld_pid_file_path"
      kill -HUP $mysqld_pid && log_success_msg "Reloading service MySQL"
      touch "$mysqld_pid_file_path"
    else
      log_failure_msg "MySQL PID file could not be found!"
      exit 1
    fi
    ;;
  'status')
    # First, check to see if pid file exists
    if test -s "$mysqld_pid_file_path" ; then
      read mysqld_pid < "$mysqld_pid_file_path"
      if kill -0 $mysqld_pid 2>/dev/null ; then
        log_success_msg "MySQL running ($mysqld_pid)"
        exit 0
      else
        log_failure_msg "MySQL is not running, but PID file exists"
        exit 1
      fi
    else
      # Try to find appropriate mysqld process
      mysqld_pid=`pidof $libexecdir/mysqld`

      # test if multiple pids exist
      pid_count=`echo $mysqld_pid | wc -w`
      if test $pid_count -gt 1 ; then
        log_failure_msg "Multiple MySQL running but PID file could not be found ($mysqld_pid)"
        exit 5
      elif test -z $mysqld_pid ; then
        if test -f "$lock_file_path" ; then
          log_failure_msg "MySQL is not running, but lock file ($lock_file_path) exists"
          exit 2
        fi
        log_failure_msg "MySQL is not running"
        exit 3
      else
        log_failure_msg "MySQL is running but PID file could not be found"
        exit 4
      fi
    fi
    ;;
    *)
      # usage
      basename=`basename "$0"`
      echo "Usage: $basename  {start|stop|restart|reload|force-reload|status}  [ MySQL server options ]"
      exit 1
    ;;
esac

exit 0
```


** Now Switch User  to mysql and run following commands**

```
[root@puppet mysql]# su mysql
bash-4.2$  mysql_install_db --user=mysql --basedir=/usr/ --datadir=/var/lib/mysql
WARNING: Could not write to config file /usr//my.cnf: Permission denied

Installing MySQL system tables...2017-08-18 07:29:26 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2017-08-18 07:29:26 0 [Note] Ignoring --secure-file-priv value as server is running with --bootstrap.
2017-08-18 07:29:26 0 [Note] /usr//sbin/mysqld (mysqld 5.6.37) starting as process 28414 ...
2017-08-18 07:29:26 28414 [Warning] Buffered warning: Changed limits: max_open_files: 1024 (requested 5000)

2017-08-18 07:29:26 28414 [Warning] Buffered warning: Changed limits: table_open_cache: 431 (requested 2000)

2017-08-18 07:29:26 28414 [Note] InnoDB: Using atomics to ref count buffer pool pages
2017-08-18 07:29:26 28414 [Note] InnoDB: The InnoDB memory heap is disabled
2017-08-18 07:29:26 28414 [Note] InnoDB: Mutexes and rw_locks use GCC atomic builtins
2017-08-18 07:29:26 28414 [Note] InnoDB: Memory barrier is not used
2017-08-18 07:29:26 28414 [Note] InnoDB: Compressed tables use zlib 1.2.3
2017-08-18 07:29:26 28414 [Note] InnoDB: Using Linux native AIO
2017-08-18 07:29:26 28414 [Note] InnoDB: Using CPU crc32 instructions
2017-08-18 07:29:26 28414 [Note] InnoDB: Initializing buffer pool, size = 128.0M
2017-08-18 07:29:26 28414 [Note] InnoDB: Completed initialization of buffer pool
2017-08-18 07:29:26 28414 [Note] InnoDB: The first specified data file ./ibdata1 did not exist: a new database to be created!
2017-08-18 07:29:26 28414 [Note] InnoDB: Setting file ./ibdata1 size to 12 MB
2017-08-18 07:29:26 28414 [Note] InnoDB: Database physically writes the file full: wait...
2017-08-18 07:29:26 28414 [Note] InnoDB: Setting log file ./ib_logfile101 size to 48 MB
2017-08-18 07:29:27 28414 [Note] InnoDB: Setting log file ./ib_logfile1 size to 48 MB
2017-08-18 07:29:27 28414 [Note] InnoDB: Renaming log file ./ib_logfile101 to ./ib_logfile0
2017-08-18 07:29:27 28414 [Warning] InnoDB: New log files created, LSN=45781
2017-08-18 07:29:27 28414 [Note] InnoDB: Doublewrite buffer not found: creating new
2017-08-18 07:29:27 28414 [Note] InnoDB: Doublewrite buffer created
2017-08-18 07:29:27 28414 [Note] InnoDB: 128 rollback segment(s) are active.
2017-08-18 07:29:27 28414 [Warning] InnoDB: Creating foreign key constraint system tables.
2017-08-18 07:29:27 28414 [Note] InnoDB: Foreign key constraint system tables created
2017-08-18 07:29:27 28414 [Note] InnoDB: Creating tablespace and datafile system tables.
2017-08-18 07:29:27 28414 [Note] InnoDB: Tablespace and datafile system tables created.
2017-08-18 07:29:27 28414 [Note] InnoDB: Waiting for purge to start
2017-08-18 07:29:27 28414 [Note] InnoDB: 5.6.37 started; log sequence number 0
2017-08-18 07:29:28 28414 [Note] Binlog end
2017-08-18 07:29:28 28414 [Note] InnoDB: FTS optimize thread exiting.
2017-08-18 07:29:28 28414 [Note] InnoDB: Starting shutdown...
2017-08-18 07:29:29 28414 [Note] InnoDB: Shutdown completed; log sequence number 1625977
OK

Filling help tables...2017-08-18 07:29:29 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2017-08-18 07:29:29 0 [Note] Ignoring --secure-file-priv value as server is running with --bootstrap.
2017-08-18 07:29:29 0 [Note] /usr//sbin/mysqld (mysqld 5.6.37) starting as process 28436 ...
2017-08-18 07:29:29 28436 [Warning] Buffered warning: Changed limits: max_open_files: 1024 (requested 5000)

2017-08-18 07:29:29 28436 [Warning] Buffered warning: Changed limits: table_open_cache: 431 (requested 2000)

2017-08-18 07:29:29 28436 [Note] InnoDB: Using atomics to ref count buffer pool pages
2017-08-18 07:29:29 28436 [Note] InnoDB: The InnoDB memory heap is disabled
2017-08-18 07:29:29 28436 [Note] InnoDB: Mutexes and rw_locks use GCC atomic builtins
2017-08-18 07:29:29 28436 [Note] InnoDB: Memory barrier is not used
2017-08-18 07:29:29 28436 [Note] InnoDB: Compressed tables use zlib 1.2.3
2017-08-18 07:29:29 28436 [Note] InnoDB: Using Linux native AIO
2017-08-18 07:29:29 28436 [Note] InnoDB: Using CPU crc32 instructions
2017-08-18 07:29:29 28436 [Note] InnoDB: Initializing buffer pool, size = 128.0M
2017-08-18 07:29:29 28436 [Note] InnoDB: Completed initialization of buffer pool
2017-08-18 07:29:29 28436 [Note] InnoDB: Highest supported file format is Barracuda.
2017-08-18 07:29:29 28436 [Note] InnoDB: 128 rollback segment(s) are active.
2017-08-18 07:29:29 28436 [Note] InnoDB: Waiting for purge to start
2017-08-18 07:29:29 28436 [Note] InnoDB: 5.6.37 started; log sequence number 1625977
2017-08-18 07:29:29 28436 [Note] Binlog end
2017-08-18 07:29:29 28436 [Note] InnoDB: FTS optimize thread exiting.
2017-08-18 07:29:29 28436 [Note] InnoDB: Starting shutdown...
2017-08-18 07:29:31 28436 [Note] InnoDB: Shutdown completed; log sequence number 1625987
OK

To start mysqld at boot time you have to copy
support-files/mysql.server to the right place for your system

PLEASE REMEMBER TO SET A PASSWORD FOR THE MySQL root USER !
To do so, start the server, then issue the following commands:

  /usr//bin/mysqladmin -u root password 'new-password'
  /usr//bin/mysqladmin -u root -h puppet.vagrant.cast.com password 'new-password'

Alternatively you can run:

  /usr//bin/mysql_secure_installation

which will also give you the option of removing the test
databases and anonymous user created by default.  This is
strongly recommended for production servers.

See the manual for more instructions.

You can start the MySQL daemon with:

  cd /usr ; /usr//bin/mysqld_safe &

You can test the MySQL daemon with mysql-test-run.pl

  cd mysql-test ; perl mysql-test-run.pl

Please report any problems at http://bugs.mysql.com/

The latest information about MySQL is available on the web at

  http://www.mysql.com

Support MySQL by buying support/licenses at http://shop.mysql.com

WARNING: Could not copy config file template /usr//share/mysql/my-default.cnf to
/usr//my.cnf, may not have access rights to do so.
You may want to copy the file manually, or create your own,
it will then be used by default by the server when you start it.

WARNING: Default config file /etc/my.cnf exists on the system
This file will be read by default by the MySQL server
If you do not want to use this, either remove it, or use the
--defaults-file argument to mysqld_safe when starting the server

bash-4.2$ mysqld_safe --user=mysql --basedir=/usr/ --datadir=/var/lib/mysql
170818 07:30:29 mysqld_safe Logging to '/var/log/mysqld.log'.
170818 07:30:29 mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql



^Z
[1]+  Stopped                 mysqld_safe --user=mysql --basedir=/usr/ --datadir=/var/lib/mysql
bash-4.2$ bg
[1]+ mysqld_safe --user=mysql --basedir=/usr/ --datadir=/var/lib/mysql &
bash-4.2$ mysql_secure_installation



NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MySQL
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MySQL to secure it, we'll need the current
password for the root user.  If you've just installed MySQL, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none):
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
Enter current password for root (enter for none):
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MySQL
root user without the proper authorisation.

Set root password? [Y/n] Y
New password:
Re-enter new password:
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MySQL installation has an anonymous user, allowing anyone
to log into MySQL without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] Y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] n
 ... skipping.

By default, MySQL comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] n
 ... skipping.

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] Y
 ... Success!




All done!  If you've completed all of the above steps, your MySQL
installation should now be secure.

Thanks for using MySQL!


Cleaning up...

```
