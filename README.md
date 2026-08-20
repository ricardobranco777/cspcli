# cspcli

Thin wrapper scripts that run the AWS CLI and Azure CLI from their official
container images, instead of installing them on the host. Each script mounts
your local config directory (`~/.aws` or `~/.azure`) into the container so
credentials and settings persist between runs, and also mounts your current
working directory at `/work` so commands that read or write local files
(`aws s3 cp`, `az storage blob upload --file`, `--template-file`, etc.) work
as expected.

## Requirements

- `docker` or `podman` (`podman` is preferred if both are present; override
  with `RUNTIME=docker` or `RUNTIME=podman`)
- Rootless container support (unprivileged user namespaces), or `sudo`

## Installation

Copy `az` and `aws` somewhere on your `PATH` and make them executable:

```console
$ install -m 755 az aws ~/.local/bin/
```

## Usage

Run either script exactly as you would the real CLI — arguments are passed
straight through to the container:

```console
$ az --version
$ aws --version
```

## Configuring `.azure`

The `az` script creates `~/.azure` (mode `700`) on first run if it doesn't
exist, and maps it into the container as `AZURE_CONFIG_DIR=/.azure`, mounted
read-write (Azure CLI needs to write tokens and cache files there).

Interactive browser login is awkward from inside a container, so if you
have a service principal (`client_id`, `client_secret`, `tenant_id`) and a
`subscription_id`, log in non-interactively instead:

```console
$ az login --service-principal \
    -u "$AZURE_CLIENT_ID" \
    -p "$AZURE_CLIENT_SECRET" \
    --tenant "$AZURE_TENANT_ID"
$ az account set --subscription "$AZURE_SUBSCRIPTION_ID"
```

The resulting config, credential cache, and tokens are written to
`~/.azure` on the host, so subsequent commands reuse the same session:

```console
$ az account show
$ az group list
```

## Configuring `.aws`

The `aws` script creates `~/.aws` (mode `700`) on first run if it doesn't
exist, and mounts it read-write into the container, with:

- `AWS_CONFIG_FILE=/.aws/config`
- `AWS_SHARED_CREDENTIALS_FILE=/.aws/credentials`

Configure it the normal way, through the wrapper:

```console
$ aws configure
```

or by creating `~/.aws/config` and `~/.aws/credentials` directly on the
host:

```console
$ mkdir -m 700 -p ~/.aws
$ cat > ~/.aws/credentials <<'EOF'
[default]
aws_access_key_id = YOUR_ACCESS_KEY_ID
aws_secret_access_key = YOUR_SECRET_ACCESS_KEY
EOF
$ cat > ~/.aws/config <<'EOF'
[default]
region = us-east-1
output = json
EOF
```

Then use the wrapper normally:

```console
$ aws sts get-caller-identity
$ aws s3 ls
```

To use a named profile, set `AWS_PROFILE` before invoking the script — the
wrapper forwards it into the container when set:

```console
$ AWS_PROFILE=myprofile aws s3 ls
```

## Working with local files

Both scripts mount the current working directory into the container at
`/work` and set it as the working directory, so relative local paths work
as-is:

```console
$ aws s3 cp ./report.csv s3://my-bucket/
$ aws s3 cp s3://my-bucket/report.csv ./
$ az storage blob upload --file ./report.csv ...
$ az deployment group create --template-file ./main.bicep ...
```

Paths outside the current directory (e.g. `../other-dir/file`, or absolute
paths elsewhere on the host) are not visible inside the container; `cd`
into the relevant directory first, or mount it yourself by editing the
script's `RUNTIME_OPTS`.

## Environment variables

| Variable  | Applies to  | Description                                              |
|-----------|-------------|-----------------------------------------------------------|
| `IMAGE`   | `az`, `aws` | Container image to run (defaults to the latest official image) |
| `RUNTIME` | `az`, `aws` | Container runtime to use (`docker` or `podman`)          |
| `SUDO`    | `az`, `aws` | Command used to gain privileges when rootless containers aren't available |

Each script also sources its own site config before running, if present
(`/etc/sysconfig/az` for `az`, `/etc/sysconfig/aws` for `aws`), so site-wide
defaults for these variables can be set there.

## See also

- [AWS CLI documentation](https://docs.aws.amazon.com/cli/latest/)
- [Azure CLI documentation](https://learn.microsoft.com/cli/azure/)
