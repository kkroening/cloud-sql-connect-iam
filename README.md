# cloud-sql-connect-iam

```
cloud-sql-connect-iam: Connect to Google Cloud SQL securely with IAM auth.

Usage:
  cloud-sql-connect-iam <instance_name|connection_name> [--] [mysql_arguments...]

Description:
  Connects to a Cloud SQL database instance over cloud-sql-proxy using a unix
  domain socket, with suppport for IAM-based authentication

  Only MySQL is currently supported.

  This tool addresses some of the limitations of the official `gcloud sql
  connect`: IAM authentication, unix-domain sockets, and accepting additional
  `mysql` command arguments.  The primary focus is to simplify automation of
  Cloud SQL provisioning for deployment from GitHub Actions with Terraform and
  workload identity federation, but it can also be used from a system
  administration standpoint to quickly, conveniently, and securely connect to a
  Cloud SQL instance without requiring use of shared secrets (username/
  password), and instead leveraging Google Cloud's IAM.

  Use of Cloud SQL IAM authentication requires care from a security standpoint
  because cloud-sql-proxy and gcloud-sql-connect use TCP sockets by default,
  which creates a local port that provides direct SQL access with only a
  username - i.e. unacceptable.  Instead, unix-domain sockets are far more
  secure because they can be locked down with 0700 file system permissions.
  Unfortunately, gcloud-sql-connect doesn't support unix-domain sockets (yet?),
  and cloud-sql-proxy requires painstaking management of temp directories to
  avoid collision with concurrently running proxies.  This tool eliminates the
  hassle by automatically running the proxy in the background with a unique
  tmpdir, and cleaning up upon exiting.

  IAM authentication is not strictly required; although it's ideal in many
  cases, there are certain cases where you can't avoid the need for password-
  based login.  In particular, creating the first IAM user for a Cloud SQL
  instance requires use of a built-in (password-based) user to bootstrap the
  IAM user's permissions.  By default, gcloud application-default credentials
  are used for IAM auth, but `--password` or `--password=...` can be
  passed to use password-based login instead.

Positional arguments:
  <instance_name|connection_name>
        The name of the Google Cloud SQL instance or full connection name.
  [--]
        If specified, pass all subsequent arguments directly to MySQL.
  [mysql_arguments...]
        Any additional arguments passed directly to the MySQL client.

Keyword arguments:
  --help
        Show help.
  --mysql-command=mysql 
        Path to the mysql command.
  --password[=...]
        Use password-based login instead of IAM auth.
  --project=...
        Google Cloud project name; defaults to gcloud default project, and
        ignored if connection_name is used.
  --region=...
        Google Cloud region (e.g. `us-central1`); defaults to gcloud default
        region, and ignored if connection_name is used.
  --proxy-command=cloud-sql-proxy
        Path to the cloud-sql-proxy command.
  --proxy-socket-dir=/tmp
        Directory to store the proxy socket.
  --proxy-timeout=3
        Time in seconds to wait for cloud-sql-proxy to start.

Examples:

  - Connect to an instance in the default gcloud project with IAM auth:

      cloud-sql-connect-iam my-instance

  - Connect and execute SQL command (e.g. via Terraform):

      cloud-sql-connect-iam my-instance -e 'SELECT "Hello world";'

  - Connect using a full connection name with IAM auth:

      cloud-sql-connect-iam my-project:us-west2:my-instance

  - Connect with password-based auth and prompt for password on stdin:

      cloud-sql-connect-iam my-instance --password

  - Connect with password-based auth and send password via environment (not
    recommended, but sometimes you have no choice):

      MYSQL_PWD="somepass" cloud-sql-connect-iam my-instance --password

  - Connect with password-based auth and send password via command-line
    argument (definitely not recommended):

      cloud-sql-connect-iam my-instance --password="somepass"

  - Connect with service account impersonation:

      (TODO: implement)
```
