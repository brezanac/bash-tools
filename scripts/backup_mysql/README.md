# backup_mysql

A highly customizable and feature rich bash script for performing backups of MySQL databases and user privileges (grant tables).

## Features

* Support for both compressed (`.sql.gz`) and uncompressed (`.sql`) output.
* Support for external option files (`mysql_cnf`) and command line parameters.
* Full support for MySQL 5+ servers, including MySQL 5.7+ specific syntax.
* Support for preserving only a specified number of recent backups (`nr_backups_kept`).
* Option to directly specify the list of databases to backup (`backup_dbs`) or query the list of available databases from the MySQL server.
* Exclusion of specific databases from the backup (`excluded_dbs`).
* Support for exporting user privileges (grant tables).
* Exclusion of specific users from exporting their user privileges (`excluded_users`).

## Requirements

This tool has the following requirements:

* **bash** - tested with version 4, no guarantee that it will run with older versions ([bash](https://www.gnu.org/software/bash/))
* **mysql** - the standard MySQL client ([mysql](https://dev.mysql.com/doc/refman/en/mysql.html))
* **mysqldump** - exports database data as SQL ([mysqldump](https://dev.mysql.com/doc/refman/en/mysqldump.html))
* **mysqlshow** - provides various information about MySQL server object properties ([mysqlshow](https://dev.mysql.com/doc/refman/en/mysqlshow.html))

## Usage

Rename `.config-example` to `.config` and adjust configuration options as needed.

Specifying `backup_path` and at least one of either `mysql_cnf` or `conn` is required. All other configuration options are optional.

If you are using an external options file (recommended) you can simply rename `.mysql_cnf-example` to `.mysql_cnf`, which will be used by default, and adjust it to fit your needs.

### WARNING! 

Even though this tool allows the use of command line parameters (`conn`) for MySQL server authentication it is **strongly advised** to use an external MySQL options file (`mysql_cnf`) instead.

Command line parameters are provided as a fallback solution for servers with non-standard configurations and should be used **only** as a last resort because they are inherently less secure than option files.

If you do use command line parameters the standard error output (`stderr`) will be suppressed in order to prevent output flood from mysql tools which will complain about using command line parameters every time the script invokes them.

```
# required
backup_path=''

# required if 'conn' is not specified 
mysql_cnf=''

# required if 'mysql_cnf' is not specified
declare -A conn=(
    [host]=''
    [user]=''
    [password]=''
)
```

## Configuration

List of configurable options. For more details on how to use these please take a look at the [examples](#examples) section.

| Property | Type | Default value | Description |
| --- | --- | --- | --- |
| **backup_path** | *string* | '' | Backup destination path. Specifies where the backups will be stored. |
| **mysql_cnf** | *string* | '' | Path to the external MySQL options file, in case one is used (**recommended**). If empty the script will fall back to using command line parameters from `$conn` (**not recommended**). |
| **conn** | *array* | empty | MySQL server connection parameters. Used if no external options file (`$mysql_cnf`) is specified. |
| **backup_dbs** | *array* | empty | List of databases to back up. This list, if specified, will take precedence over the one obtained from the MySQL server. |
| **excluded_dbs** | *array* | empty | List of databases to exclude from the backup. |
| **nr_backups_kept** | *integer* | 5 | Maximum number of backups per database to preserve. Use `false` for infinite. Backups are sorted chronologically by time of creation, from newest to oldest, before deletion. Oldest backups are deleted first. |
| **compress_backup** | *boolean* | true | Use compression (gzip) for output. Allowed values: `true`, `false`.  |
| **export_grants** | *boolean* | false | Export user privileges (grant tables) as a separate backup. Allowed values: `true`, `false`. |
| **excluded_users** | *array* | empty | List of MySQL users which will be excluded from the user privileges export. Items should be in the format `"'user'@'host'"` (double quotes are **required**), one item per line. |
| **grants_path** | *string* | '' | User privileges backup destination path. Specifies where the user privileges (grant tables) exports should be stored. |
| **rnd_token_length** | *integer* | 8 | Length of the random token. Random tokens are used to make sure backup filenames are truly unique. |
| **date_format** | *string* | '+%d.%m.%Y %H:%M:%S' | General date and time format used in various places. Please consult `man date` or visit the [online version](http://man7.org/linux/man-pages/man1/date.1.html) for further details on how to customize this option. |

## Examples

### backup_path

`backup_path` can use both absolute paths and those relative to the script.

```
# absolute path
backup_path='/backup'

# relative path
backup_path='../backup'
```

### mysql_cnf

Detailed explanation about what an external options file is and how to use one can be found in the official MySQL documentation [here](https://dev.mysql.com/doc/refman/en/option-files.html).

The external options file can contain one `[client]` and multiple `[client_dbn]` sections. 

The `_dbn` parameter is the value of the `--default-group-suffix` ([link](https://dev.mysql.com/doc/refman/en/option-file-options.html#option_general_defaults-group-suffix)) option which is accepted by most standard MySQL command line tools.

```
[client]
host=
user=
password=

[client_dbn]
user=
password=
```

The following example can be used if only one MySQL user is required to access all databases so there is no need for separate user credentials.

```
# one user has access to all databases (i.e. "backup")
[client]
host=localhost
user=backup
password=password
```

The `[client_dbn]` sections will be used for authenticating with individual databases which require separate user credentials in order to access them.

In that case you can create one `[client_dbn]` section for every database. 

If no `[client_dbn]` section is matched the default `[client]` section will be used instead.

```
# used as fallback in case no [client_dbn] section is matched
[client]
host=localhost
user=backup
password=password

# used for accessing database 'db1'
[client_db1]
user=db1_user
password=db1_password

# used for accessing database 'dbx'
[client_dbx]
user=dbx_user
password=dbx_password
```

### conn

Connect user `backup` to host `localhost` authenticated with password `password`.

```
declare -A conn=(
    [host]='localhost'
    [user]='backup'
    [password]='password'
)
```

In case `mysql_cnf` is specified this option will be ignored.

### backup_dbs

Backup only databases named `logs` and `content`.

```
backup=(
    'logs'
    'content'
)
```

If this option is left empty the list of available databases will be retrieved from the MySQL server directly for the user credentials specified during the query.

### excluded_dbs

Exclude databases `information_schema`, `mysql` and `performance_schema` from the backup.

```
excluded_dbs=(
    'information_schema'
    'mysql'
    'performance_schema'
)
```

You can use this option to fine tune the final list of databases to back up.

### nr_backups_kept

The following example will first create `/logs/logs_6.sql` and then delete the oldest backup.

```
# example folder structure of 'backup_path', sorted by creation date, 'logs_1.sql' being the oldest one
.
└───logs
    ├───logs_1.sql
    ├───logs_2.sql
    ├───logs_3.sql
    ├───logs_4.sql
    └───logs_5.sql
```

```
nr_backups_kept=5
```

```
# folder structure after script execution
.
└───logs
    ├───logs_2.sql
    ├───logs_3.sql
    ├───logs_4.sql
    ├───logs_5.sql
    └───logs_6.sql
```

### compress_backup

```
# compressed output, filenames will end with '.sql.gz'
compress_backup=true

# uncompressed output, filenames will end with '.sql'
compress_backup=false
```

### export_grants

```
# export user privileges
export_grants=true

# do not export user privileges
export_grants=false
```

User privileges will be exported separately in a path specified by `grants_path`.

### excluded_users

Exclude users `'user'@'host'` and `'user_1'@'host'` from exporting their user privileges (grant tables).

```
excluded_users=(
    "'user'@'host'"
    "'user_1'@'host'"
)
```

### grants_path

```
grants_path='/backup/mysql/grants'
```

### rnd_token_length

The following example will create a randomly generated six characters long token consisting of uppercase and lowercase letters as well as numerical digits. 

```
# adds a string with the following regex to the output filename [A-Za-z0-9]{6}
rnd_token_length=6
```

## Performance

Depending on underlying hardware and especially with a large number of files to back up or if  archive compression is used (`compress_archives` is set to `true`), this script can be very resource intensive. It is therefore recommended to combine it with tools such as [nice](https://linux.die.net/man/1/nice) and [ionice](https://linux.die.net/man/1/ionice) which can reduce the CPU and I/O usage, respectively, during script execution.

Many distributions already ship with both tools included however you can check their presence by simply running the `which` command.

```
# checking for presence of 'nice' and 'ionice'
$ which nice
/usr/bin/nice

$ which ionice
/usr/bin/ionice
```

Here is a typical example of using both `nice` and `ionice` to lower the CPU and I/O priority during execution.

```
/usr/bin/nice -n 19 /usr/bin/ionice -c2 -n7 ./backup_files
```

**NOTE:** default values for niceness vary across different operating systems which is why it's highly advised to consult the manual before adjusting their values. Additionally, increasing niceness above the default value requires superuser (root) privileges.

## License

This project is licensed under the MIT License - see the [LICENSE](../../LICENSE) file for details.