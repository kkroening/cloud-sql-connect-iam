# cloud-sql-connect-iam

```
cloud-sql-connect-iam: Connect to Google Cloud SQL securely with IAM auth.

Usage:
  cloud-sql-connect-iam <instance_name|connection_name> [ -- [subcmd...]]

Description:
  Connects to a Cloud SQL database instance over cloud-sql-proxy using a unix
  domain socket and IAM-based authentication (optional), and then runs a command
  while the proxy continues in the background.

  This tool addresses some of the limitations of the official `gcloud sql
  connect`: IAM authentication, unix-domain sockets, and accepting additional
  sub-command arguments.  The primary focus is to simplify automation of
  Cloud SQL provisioning for deployment from GitHub Actions with Terraform and
  workload identity federation, but it can also be used from a system
  administration standpoint to quickly, conveniently, and securely connect to a
  Cloud SQL instance without requiring use of shared secrets (password), and
  instead leveraging Google Cloud's IAM.

  Conceptually, this tools effectively acts like a Python async context manager
  that runs cloud-sql-proxy over a unix-domain socket in the background, runs a
  subcommand with a predictable environment, and then shuts down the proxy:

    # Pseudo code:
    async with cloud_sql_connect_iam.Context(...) as env:
        assert 'CSQL_CIAM_CONNECTION_NAME' in env
        assert 'MYSQL_HOST' in env
        # etc.

  The subcommand runs with several environment variables exported:

  *   `CSQL_CIAM_CONNECTION_NAME`
  *   `CSQL_CIAM_GCP_PROJECT`
  *   `CSQL_CIAM_GCP_REGION`
  *   `CSQL_CIAM_IAM_AUTH`
  *   `CSQL_CIAM_IMPERSONATE_SERVICE_ACCOUNT`
  *   `CSQL_CIAM_INSTANCE_NAME`
  *   `CSQL_CIAM_PROXY_SOCKET_FILE`
  *   `CSQL_CIAM_USER`
  *   `LIBMYSQL_ENABLE_CLEARTEXT_PLUGIN`
  *   `MYSQL_UNIX_PORT`

IAM authentication:
  Use of Cloud SQL IAM authentication requires care from a security standpoint
  because cloud-sql-proxy and gcloud-sql-connect use TCP sockets by default,
  which creates a local port that provides direct SQL access with only a
  username - i.e. unacceptable.  Instead, unix-domain sockets are far more
  secure because they can be locked down with 0700 file system permissions.
  Unfortunately, gcloud-sql-connect doesn't support unix-domain sockets
  (yet?), and cloud-sql-proxy requires painstaking management of temp
  directories to avoid collision with concurrently running proxies.  This tool
  eliminates the hassle by automatically running the proxy in the background
  with a unique tmpdir, and cleaning up upon exiting.

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
  [-- [subcmd...]]
        If specified, `--` demarcates the following arguments as being the
        sub-command to run within the scope of cloud-sql-proxy running in the
        background.  If unspecified, the sub-command defaults to `mysql`.

Keyword arguments:
  --help
        Show help.
  --impersonate-service-account=...
        Comma separated list of service accounts to impersonate. Last value
        is the target account.
  --no-iam
        Use password-based login instead of IAM auth.
  --project=...
        Google Cloud project name; defaults to gcloud default project, and
        ignored if connection_name is used.
  --region=...
        Google Cloud region (e.g. `us-central1`); defaults to gcloud default
        region, and ignored if connection_name is used.
  --proxy-command=cloud-sql-proxy
        Path to the cloud-sql-proxy command.
  --proxy-tmpdir=/tmp
        Directory to store the proxy socket.
  --proxy-timeout=3
        Time in seconds to wait for cloud-sql-proxy to start.

Examples:

  - Connect to an instance in the default gcloud project+region with IAM auth:

      cloud-sql-connect-iam my-instance

  - Connect and execute MySQL command (e.g. from Terraform):

      cloud-sql-connect-iam my-instance --         mysql --user="someuser" --execute='SELECT "Hello world";'

  - Connect using a full connection name with IAM auth:

      cloud-sql-connect-iam my-project:us-west2:my-instance

  - Connect with password-based auth and prompt for password on stdin:

      cloud-sql-connect-iam my-instance --no-iam

  - Connect with password-based auth and send password via environment (not
    recommended, but sometimes you have no choice - e.g. Terraform local-exec):

      MYSQL_PWD="somepass" cloud-sql-connect-iam my-instance --no-iam

  - Connect with password-based auth and send password via command-line
    argument (definitely not recommended):

      cloud-sql-connect-iam my-instance --no-iam --         mysql --user="someuser" --password="somepass"

  - Connect with service account impersonation:

      cloud-sql-connect-iam my-instance         --impersonate-service-account=svc@proj.iam.gserviceaccount.com

TODO/FIXME/Warning:
  Automatic username inference currently only happens if no explicit subcommand
  is provided.  This makes `cloud-sql-connect-iam` behave like `gcloud sql connect`
  in the default case by providing a SQL command prompt based entirely on the
  user's gcloud configuration (or `--impersonate-service-account=...` if
  specified), but no such convenience happens if you pass any explicit
  subcommand - even if it's simply `-- mysql`.  Making this more consistently
  convenient is a possible future improvement.
```
