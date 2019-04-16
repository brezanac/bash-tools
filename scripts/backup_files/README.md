# backup_files

A highly customizable and feature rich script for performing backups of files and folders.

## Features

* Support for two backup methods: syncing (requires `rsync`) and archiving (requires `tar`). 
* Archived backups can produce both compressed (`gzip`) and uncompressed output.
* Support for preserving only a specified number of recent backups (`$nr_backups_kept`).
* Specifying source paths directly (`source_direct_paths`) or as a list of paths whose immediate children (`source_parent_paths`) will be backed up.
* Excluding specific paths from the backup (`excluded_paths`).
* Notification emails on failed backups etc.
* Detailed output for all operations.

## Requirements

This tool has the following requirements:

* **bash** - tested with version 4, no guarantee that it will run with older versions ([bash](https://www.gnu.org/software/bash/))
* **rsync** - for performing sync based backups ([rsync](https://rsync.samba.org/))
* **tar** - for performing archive based backups ([tar](https://www.gnu.org/software/tar/manual/tar.html))
* **mail** - in case email notifications are used ([mail](https://mailutils.org/))

## Usage

Rename `.config-example` to `.config` and adjust configuration options as needed.

Specifying `backup_path` and at least one of either `source_direct_paths` or `source_parent_paths` is required. All other configuration options are optional.

```
# required
backup_path='/path/to/backup/destination'

# required if 'source_parent_paths' is empty
source_direct_paths=(
    '/backup/source/path_1'
    '...'
)

# required if 'source_direct_paths' is empty
source_parent_paths=(
    '/backup/source/parent_path_1'
    '...'
)

# 'source_direct_paths' and 'source_parent_paths' can be combined
```

## Configuration

List of configurable options which can be adjusted in the external configuration file (`.config`). For more details on how to use these please take a look at the [examples](#examples) section.

| Property | Type | Default value | Description |
| --- | --- | --- | --- |
| **backup_path** | *string* | '' | Backup destination path. Specifies where the backups will be stored. |
| **source_direct_paths** | *array* | empty | List of paths to backup. Each item will be backed up separately. |
| **source_parent_paths** | *array* | empty | List of paths whose immediate children will be backed up. Each item will be backed up separately. |
| **excluded_paths** | *array* | empty | List of paths to exclude during backup. Both absolute and basename paths are supported, as well as glob based patterns. |
| **nr_backups_kept** | *integer* | 5 | Maximum number of backups to preserve. Use `false` for infinite. Backups are sorted chronologically by time of creation, from newest to oldest, before deletion. Oldest backups are deleted first. |
| **backup_method** | *string* | 'archive' | Backup method. Allowed values: `sync`, `archive`. In case `sync` is used source paths will be one-way synced to `backup_path` with `rsync`. Backups made with `archive` will be stored as archives using `tar`. |
| **compress_archives** | *boolean* | false | Use compression (gzip) for archives. Allowed values: `true`, `false`. Works only if `backup_method` is set to `archive`. Please note that depending on the used hardware and the amount of data to back up, compression could significantly increase backup time (see [performance](#performance) section) |
| **rnd_token_length** | *integer* | 8 | Length of the random token. Random tokens are used to make sure backup file and folder names are truly unique. |
| **send_email_notifications** | *boolean* | false | Sending email notifications in case of issues. Allowed values: `true`, `false`. If set to `true` email notifications will be sent for events like insufficient space to perform backups etc. |
| **email** | *array* | - / - | Configuration for email notifications. Required only if `send_email_notifications` is set to `true`.<br> <ul><li>`from` - sender , content of the 'From' header</li><li>`to` - receiver, content of the 'To' header</li><li>`reply_to` - reply email, content of the 'Reply-To' header</li></ul> |
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

### source_direct_paths

`source_direct_paths` provides an excellent way to have full control over backup sources.

The following example will back up `/home/user_1`, `/home/user_2` and `/home/user_4`.

```
source_direct_paths=(
    '/home/user_1'
    '/home/user_2'
    '/home/user_4'
)
```

### source_parent_paths

`source_parent_paths` provides an option to specify parent folders whose immediate children will be backed up.

The following example will backup `/home/dewey`, `/home/huey`, `/home/louie`, `/var/backup_this`, `/var/this_too` but **NOT** `/home` or `/var`.

```
# example folder structure
.
├───home
│   ├─── dewey
│   ├─── huey
│   └─── louie
└───var
    ├─── backup_this
    └─── this_too
```

```
source_parent_paths=(
    '/home'
    '/var'
)
```

`source_direct_paths` and `source_parent_paths` are **NOT** mutually exclusive. You can combine them together to get the final list of paths to backup. You can also use `excluded_paths` to exclude specific paths from any final list of source paths.

### excluded_paths

Excluding `/var/www/htdocs` and all of it's content.

```
excluded_paths=(
    '/var/www/htdocs'
)
```

Excluding all paths ending with `cgi-bin` as their basename (last part of the path in it's entirety).

```
excluded_paths=(
    'cgi-bin'
)
```

Excluding all paths ending with `gi-bin` as their basename. This will **NOT** exclude `cgi-bin` because `gi-bin` doesn't cover the basename in it's entirety.

```
excluded_paths=(
    'gi-bin'
)
```

Excluding all paths ending with `.php`.

```
excluded_paths=(
    '*.php'
)
```

### nr_backups_kept

The following example will first create `/dewey/dewey_6` and then delete the oldest backup.

```
# example folder structure of 'backup_path', sorted by creation date, 'dewey_1' being the oldest one
.
└───dewey
    ├───dewey_1
    ├───dewey_2
    ├───dewey_3
    ├───dewey_4
    └───dewey_5
```

```
nr_backups_kept=5
```

```
# folder structure after script execution
.
└───dewey
    ├───dewey_2
    ├───dewey_3
    ├───dewey_4
    ├───dewey_5
    └───dewey_6
```

**NOTE:** to prevent data loss cleanup is performed only after a successful backup of the item which means there needs to be at least enough available space in `backup_path` to accommodate `nr_backups_kept + 1` backups.

### backup_method

```
# source paths will be copied with 'rsync'
backup_method='sync'
```

```
# source paths will be archived with 'tar'
backup_method='archive'
```

### compress_archives

```
# 'backup_method' needs to be set to 'archiving' in order to use compression
backup_method='archiving'

# archived backups will be compressed with gzip
compress_archives=true
```

### rnd_token_length

The following example will create a randomly generated six characters long token consisting of uppercase and lowercase letters as well as numerical digits. 

```
# adds a string with the following regex to the output filename [A-Za-z0-9]{6}
rnd_token_length=6
```

### send_email_notifications

```
# email notifications will be sent on failure
send_email_notifications=true

# no email notifications will be sent on failure
send_email_notifications=false
```

### email

The following email configuration is an example of sending an email to `admin@example.com` from `backup@example.com` with a reply address to `replies@example.com` in case there is an issue during the backup process.

```
declare -A email=(
    [from]='backup@example.com'
    [to]='admin@example.com'
    [reply_to]='replies@example.com'
)
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