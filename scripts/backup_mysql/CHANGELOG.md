# Change Log (backup_mysql)

This project adheres to [Semantic Versioning](http://semver.org/).

## 1.0.0

### Added

* Support for both compressed (.sql.gz) and uncompressed (.sql) output.
* Support for both external options file and command line parameters.
* Added full support for MySQL 5+ servers, including MySQL 5.7+ specific syntax.
* Option to limit the number of preserved backups.
* Option to directly specify a list of databases to backup or query the server for a list of available databases.
* Option to exclude specific databases from backing up.
* Support for exporting user privileges (grant tables).
* Option to exclude specific users from being exported during user privileges backup.
