>  ⚠️ This is forked so that we can patch it to allow the sign-in flow to interact with Okta

<p align="center">
    <img src="images/logo/cover.png" height="96">
  <p align="center">AWS STS credentials via Google Workspace</p>
</p>

`gsts` (short for `Google STS`) is [playwright](https://github.com/microsoft/playwright) automation to obtain and store AWS STS credentials to interact with Amazon services by authenticating against a pre-configured Google Workspace SAML instance.

This allows you to configure AWS to rely on Google Workspace as your identity provider, moving the responsibility away from Amazon into Google to validate your login credentials. This is a widly popular solution when looking to offer Single-Sign On capabilities inside organizations.

The problem is that this flow is tailored for the web which makes command-line usage a lot more difficult. This utility is helper around that.

#### Features:

* First-time only headful design for interactively entering your Google Workspace credentials.
* Full support for all 2FA methods as provided by Google, including security keys.
* Persistent headless re-authentication system.
* Supports custom session durations (from 15min to 12h).
* Compatible with Amazon ECR.
* Daemon helper for continously refreshing the STS token (only available on macOS for now).
* Offers a quick action to open the AWS console from the command-line.
* Support for AWS China (`aws-cn`) and AWS GovCloud (US) (`aws-us-gov`) ARNs.

## Installation

### macOS

```shell
brew tap ruimarinho/tap
brew install gsts
```

### Other Platforms

Install the package via `npm`:

```sh
npm install --global gsts
```

or via `yarn`:

```
yarn global add gsts
```

## Usage

There are three key options or variables you need know about (you can read more about how to discover them below):

1. Google's Identity Provider ID, or IDP ID.
1. Google's Service Provider ID, or SP ID.
1. AWS role ARN to authenticate with.

You can then launch `gsts` using command-line options:

```sh
gsts --aws-role-arn arn:aws:iam::123456789:role/foobar --sp-id 12345 --idp-id A12bc34d5 --username foo@bar.com
```

Alternatively, you can use environment variables instead:

```sh
GOOGLE_USERNAME=foo@bar.com GOOGLE_SP_ID=12345 GOOGLE_IDP_ID=A12bc34d5 gsts
```

That's it! The first authentication will be performed directly on a headful browser where all of the authentication challenges generated by Google are natively supported (TOTP, Push, SMS, Security Keys, etc). Subsequent runs use an existing session to obtain fresh STS credentials every time the utility is executed.

To make sure the profile generated by `gsts` - by default, called `sts` - is used on other tools interacting with AWS services via STS tokens (`aws`, `kubectl`, etc.), make sure `AWS_PROFILE=sts` is set as an environment variable. Alternatively, you can force `gsts` to use the _default_ profile name by using `gsts --aws-profile=default`.

### `credential_process`

`gsts` can be invoked as a credential source through [`credential_process`](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sourcing-external.html) with the `--json` option.

For example, add this to your `~/.aws/config` file:

```
[profile sts]
credential_process = gsts --aws-role-arn arn:aws:iam::123456789:role/foobar --sp-id 12345 --idp-id A12bc34d5 --json
```

## Amazon ECR

If you'd like to automatically authenticate your Docker installation before pulling private images from Amazon ECR, you can use the fantastic [ECR Docker Credential Helper](https://github.com/awslabs/amazon-ecr-credential-helper) in combination with `gsts`.

1. Install `docker-credential-helper-ecr` (on macOS, you can do it via Homebrew using `brew install docker-credential-helper-ecr`).
2. Create a file named `docker-credential-gsts` which needs to be on your `PATH` environment variable. To keep things organize, let's create it inside `~/.docker/bin/` and add that folder to `PATH`.

    ```sh
    #!/bin/sh
    gsts < /dev/tty > /dev/tty
    exec docker-credential-ecr-login $@
    ```

    Make sure the file is executable with `chmod +x ~/.docker/bin/docker-credential-gsts`.

3. Add the following config to your `~/.docker/config.json` file, provided `gsts` is in your `PATH`:

    ```json
    {
      "credHelpers" : {
        "<ACCOUNT_ID>.dkr.ecr.<ECR_REGION>.amazonaws.com" : "gsts"
      }
    }
    ```

4. Depending on your setup, you may need to specify an additional environment variable `AWS_DEFAULT_REGION` (e.g. `us-east-1`).

The next step a `docker pull` for an image from an ECR registry matching the string above is called, Docker will invisibly call `gsts` and perform authentication on your behalf.

## Daemon

If you are a heavy Amazon AWS user with a constant need of a fresh STS token or if you find the maximum amount of time a session can live by Amazon's own rules too short (12 hours), you can setup a helper to periodically call `gsts` for you.

### macOS

`gsts` comes with a basic LaunchAgent plist generator which it will try to copy to `~/Library/LaunchAgent/io.ruimarinho.gsts.plist` and automatically load.

The `gsts` helper doesn't actually run in background. The OS native scheduler will periodically (every 10min) execute `gsts` for you to make sure a fresh STS token is available whenever you need it.

Simply call `gsts` with the daemon option to install the helper:

```sh
gsts --daemon
```

If `/usr/local/var/log/` is not user-writable, you may create that directory or customize the path for logs using `--daemon-out-log-path` and `--daemon-error-log-path`.

You may safely disable the helper at any time by unloading:

```
launchctl unload ~/Library/LaunchAgents/io.ruimarinho.gsts.plist
```

## Quick Actions

`gsts` offer a quick way to open the Amazon AWS console via the command line:

```sh
gsts console
```

## Reference

```sh
❯ gsts --help

Commands:
  index.js console  Authenticate via SAML and open Amazon AWS console in the
                    default browser

Options:
  --help                         Show help                             [boolean]
  --version                      Show version number                   [boolean]
  --aws-profile                  AWS profile name for storing credentials
                                                                [default: "sts"]
  --aws-role-arn                 AWS role ARN to authenticate with
  --aws-session-duration         AWS session duration in seconds (defaults to
                                 the value provided by the IDP, if set) [number]
  --aws-shared-credentials-file  AWS shared credentials file
                                 [default: "/Users/ruimarinho/.aws/credentials"]
  --clean                        Start authorization from a clean session state
  --daemon                       Install daemon service (only on macOS for now)
  --daemon-out-log-path          Path for storing the output log of the daemon
                                 [default: "/usr/local/var/log/gsts.stdout.log"]
  --daemon-error-log-path        Path for storing the error log of the daemon
                                 [default: "/usr/local/var/log/gsts.stderr.log"]
  --json                         JSON output (compatible with AWS config's
                                 credential_process)
  --force                        Force re-authorization even with valid session
  --idp-id, --google-idp-id      Google Identity Provider ID (IDP ID) [required]
  --engine                       Set custom browser engine
                [choices: "chromium", "firefox", "webkit"] [default: "chromium"]
  --engine-executable-path       Set custom executable path for browser engine
                                                                 [default: null]
  --sp-id, --google-sp-id        Google Service Provider ID (SP ID)   [required]
  --username, --google-username  Google username to auto pre-fill during login
  -v, --verbose                  Log verbose output                      [count]
```

For compatibility reasons, most environment variables supported [aws-google-auth](https://github.com/cevoaustralia/aws-google-auth) are also supported by `gsts`:

| Description | Command-Line Option | Env Variable | Required |
|-------------|---------------------|--------------|----------|
| Google IDP ID | `--idp-id` | `$GOOGLE_IDP_ID` | Yes |
| Google SP ID | `--sp-id` | `$GOOGLE_SP_ID` | Yes |
| Google Username | `--username` | `$GOOGLE_USERNAME` | No |
| AWS Shared Credentials File | `--aws-shared-credentials-file` | `$AWS_SHARED_CREDENTIALS_FILE` | No (_default: ~/.aws/credentials_) |
| AWS Profile | `--aws-profile` | `$AWS_PROFILE` | No (_default: default_)

## Discovery of IDP and SP IDs

If you're the admin of Google Workspace, after configuring the SAML application for AWS you can extract the SP ID by looking at the `service` parameter of the SAML AWS application page.

<img src="images/google-workspace-sp-id.png" width="800px">

The IDP ID can be found under _Security > Set up single sign-on (SSO) for SAML applications_ as the parameter `idpid`.

<img src="images/google-workspace-idp-id.png" width="800px">

In case you are using a pre-configured AWS SAML application as traditionally available under the dotted menu on any Google app (Gmail, Calendar and so on) you can instead right-click the AWS icon and copy the link:

<img src="images/google-workspace-aws-app.png" width="300px">

The copied URL will be in the format of `https://accounts.google.com/o/saml2/initsso?idpid=<IDP_ID>&spid=<SP_ID>&forceauthn=false`.

## Troubleshooting

**gsts conflicts with an alias from oh-my-zsh's git plugin**

[ohmyzsh's git plugin](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/git) includes an alias named `gsts` as a shorthand for `git stash show --text`. You can either disable the `git` plugin entirely or, alternatively, add `unalias gsts` at the end of your dotfiles if you don't use this git command often.

**"Error when retrieving credentials from custom-process: Error: Failed to launch the browser process!" when using the aws-cli with credential_process**

Although seamingly unrelated to `gsts`, try unsetting `LD_LIBRARY_PATH` before calling it, like so:

```bash
credential_process = bash -c "unset LD_LIBRARY_PATH; gsts --aws-role-arn arn:aws:iam::123456789:role/foobar --sp-id 12345 --idp-id A12bc34d5 --username foo@bar.com --json"
```

## License

MIT
