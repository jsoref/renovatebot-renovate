# Self-Hosting Renovate

## Installing Renovate OSS CLI

#### npmjs

```sh
$ npm install -g renovate
```

Since renovate v20 `npm`, `pnpm` and `yarn` are no longer embedded, so you need to install them globally if you need to update lockfiles.

```sh
$ npm install -g yarn pnpm
```

The same goes for any other third party binary tool that may be needed, such as `gradle` or `poetry` - you need to make sure they are installed and the appropriate version you need before running Renovate.

#### Docker

Renovate is available for Docker via an automated build [renovate/renovate](https://hub.docker.com/r/renovate/renovate/). It builds `latest` based on the `master` branch and all semver tags are published too. All the following are valid:

```sh
$ docker run --rm renovate/renovate
$ docker run --rm renovate/renovate:19.181.2
$ docker run --rm renovate/renovate:19.181
$ docker run --rm renovate/renovate:19
```

(Please look up what the latest actual tags are though, do not use the above literally).

If you wish to configure Renovate using a `config.js` file then map it to `/usr/src/app/config.js` using Docker volumes. For example:

```sh
$ docker run --rm -v "/path/to/your/config.js:/usr/src/app/config.js" renovate/renovate
```

#### Kubernetes

Renovate's official Docker image is compatible with Kubernetes. The following is an example manifest of running Renovate against a GitHub Enterprise server. First the Kubernetes manifest:

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: renovate
spec:
  schedule: '@hourly'
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: renovate
              # Update this to the latest available and then enable Renovate on
              # the manifest
              image: renovate/renovate:23.19.2
              args:
                - user/repo
              # Environment Variables
              envFrom:
                - secretRef:
                    name: renovate-env
          restartPolicy: Never
```

And also this accompanying `secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: renovate-env
type: Opaque
stringData:
  GITHUB_COM_TOKEN: 'any-personal-user-token-for-github-com-for-fetching-changelogs'
  # set to true to run on all repos you have push access to
  RENOVATE_AUTODISCOVER: 'false'
  RENOVATE_ENDPOINT: 'https://github.company.com/api/v3'
  RENOVATE_GIT_AUTHOR: 'Renovate Bot <bot@renovateapp.com>'
  RENOVATE_PLATFORM: 'github'
  RENOVATE_TOKEN: 'your-github-enterprise-renovate-user-token'
```

A `config.js` file can be added to the manifest using a `ConfigMap` as shown in the following example (using a "dry run" in github.com)

```yaml
---
 apiVersion: v1
kind: ConfigMap
metadata:
  name: renovate-config
data:
  config.json: |-
    {
      "logLevel" : "debug",
      "repositories": ["orgname/repo","username/repo"],
      "dryRun" : "true"
    }

---
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: renovate-bot
spec:
  schedule: '@hourly'
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - image: renovate/renovate:23.22.1
              name: renovate-bot
              env: # For illustration purposes, please user secrets.
                - name: RENOVATE_PLATFORM
                  value: 'github'
                - name: RENOVATE_TOKEN
                  value: 'some-token'
                - name: RENOVATE_AUTODISCOVER
                  value: 'false'
                - name: RENOVATE_BASE_DIR
                  value: '/tmp/renovate/'
                - name: RENOVATE_CONFIG_FILE
                  value: '/opt/renovate/config.json'
              volumeMounts:
                - name: config-volume
                  mountPath: /opt/renovate/
                - name: work-volume
                  mountPath: /tmp/renovate/
          restartPolicy: Never
          volumes:
            - name: config-volume
              configMap:
                name: renovate-config
            - name: work-volume
              emptyDir: {}
```

## Configuration

Self-hosted Renovate can be configured using any of the following (or a combination):

- A `config.js` file (can also be named `config.json`, but you can't have both at the same time)
- CLI params
- Environment params

Note that some Renovate configuratino options are _only_ available for self-hosting, and so can only be configured using one of the above methods. These are described in the [Self-hosted Configuration](./self-hosted-configuration.md) doc.

## Authentication

Regardless of platform, you need to select a user account for `renovate` to assume the identity of, and generate a Personal Access Token. It is recommended to be `@renovate-bot` if you are using a self-hosted server with free choice of usernames.
It is also recommended that you configure `config.gitAuthor` with the same identity as your Renovate user, e.g. like `"gitAuthor": "Renovate Bot <renovate@whitesourcesoftware.com>"`.

#### GitHub Enterprise

First, [create a personal access token](https://help.github.com/articles/creating-an-access-token-for-command-line-use/) for the bot account (select "repo" permissions).
Configure it either as `token` in your `config.js` file, or in environment variable `RENOVATE_TOKEN`, or via CLI `--token=`.

#### GitLab CE/EE

First, [create a personal access token](https://docs.gitlab.com/ee/api/README.html#personal-access-tokens) for the bot account (select "api" scope).
Configure it either as `token` in your `config.js` file, or in environment variable `RENOVATE_TOKEN`, or via CLI `--token=`.
Don't forget to configure `platform=gitlab` somewhere in config.

#### Bitbucket Cloud

First, [create an AppPassword](https://confluence.atlassian.com/bitbucket/app-passwords-828781300.html) for the bot account.
Configure it as `password` in your `config.js` file, or in environment variable `RENOVATE_PASSWORD`, or via CLI `--password=`.
Also be sure to configure the `username` for your bot account too.
Don't forget to configure `platform=bitbucket` somewhere in config.

#### Bitbucket Server

Create a [Personal Access Token](https://confluence.atlassian.com/bitbucketserver/personal-access-tokens-939515499.html) for your bot account.
Configure it as `password` in your `config.js` file, or in environment variable `RENOVATE_PASSWORD`, or via CLI `--password=`.
Also configure the `username` for your bot account too, if you decided not to name it `@renovate-bot`.
Don't forget to configure `platform=bitbucket-server` somewhere in config.

#### Azure DevOps

First, [create a personal access token](https://docs.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/pats) for the bot account.
Configure it either as `token` in your `config.js` file, or in environment variable `RENOVATE_TOKEN`, or via CLI `--token=`.
Don't forget to configure `platform=azure` somewhere in config.

### Gitea

First, [create a access token](https://docs.gitea.io/en-us/api-usage/#authentication-via-the-api) for your bot account.
Configure it as `token` in your `config.js` file, or in environment variable `RENOVATE_TOKEN`, or via CLI `--token=`.
Don't forget to configure `platform=gitea` somewhere in config.

## GitHub.com token for release notes

If you are running on any platform except github.com, it's important to also configure the environment variable `GITHUB_COM_TOKEN` containing a personal access token for github.com. This account can actually be _any_ account on GitHub, and needs only read-only access. It's used when fetching release notes for repositories in order to increase the hourly API limit. It's also OK to configure the same as a host rule instead, if you prefer that.

**Note:** If you're using renovate in a project where dependencies are loaded from github.com (such as Go m=Modules hosted on github) it is highly recommended to add a token as you will run in the rate limit from the github.com api, which will lead to renovate closing and reopening PRs because it could not get reliable info on updated dependencies.

## File/directory usage

By default, Renovate will store all files within a `renovate/` subdirectory of the operating system's temporary directory, e.g. `/tmp/renovate/`.

Repository data will be copied or cloned into unique subdirectories under `repos/`, e.g. `/tmp/renovate/repos/github/owner1/repo-a/`.

Cache data - such as Renovate's own cache as well as that for npm, yarn, composer, etc - will be stored in `/tmp/renovate/cache`.

If you wish to override the base directory to be used (e.g. instead of `/tmp/renovate/`) then configure a value for `baseDir` in `config.js`, or via env (`RENOVATE_BASE_DIR`) or via CLI (`--base-dir=`).

If you wish to override the cache location specifically then configure a value for `cacheDir` instead.

## Usage

The following example uses the Renovate CLI tool, which can be installed by running `npm i -g renovate`.

If running your own Renovate bot then you will need a user account that Renovate will run as. It's recommended to use a dedicated account for the bot, e.g. name it `renovate-bot` if on your own instance. Create and save a Personal Access Token for this account.

Create a Renovate config file, e.g. here is an example:

```js
module.exports = {
  endpoint: 'https://self-hosted.gitlab/api/v4/',
  token: '**gitlab_token**',
  platform: 'gitlab',
  logLevel: 'debug',
  onboardingConfig: {
    extends: ['config:base'],
  },
  repositories: ['username/repo', 'orgname/repo'],
};
```

Here change the `logFile` and `repositories` to something appropriate. Also replace gitlab-token value with the one created during the previous step.

If running against GitHub Enterprise, change the above gitlab values to the equivalent github ones.

You can save this file as anything you want and then use `RENOVATE_CONFIG_FILE` env variable to tell Renovate where to find it.

Most people will run Renovate via cron, e.g. once per hour. Here is an example bash script that you can point `cron` to:

```sh
#!/bin/bash

export PATH="/home/user/.yarn/bin:/usr/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:$PATH"
export RENOVATE_CONFIG_FILE="/home/user/renovate-config.js"
export RENOVATE_TOKEN="**some-token**" # GitHub, GitLab, Azure DevOps
export GITHUB_COM_TOKEN="**github-token**" # Delete this if using github.com

# Renovate
renovate
```

Note: the GitHub.com token in env is necessary in order to retrieve Release Notes that are usually hosted on github.com. You don't need to add it if you are already running the bot against github.com, but you do need to add it if you're using GitHub Enterprise, GitLab, Azure DevOps, or Bitbucket.

You should save and test out this script manually first, and add it to cron once you've verified it.

## Kubernetes for Gitlab, using Git over SSH

This section describes how to use git binary with ssh for Gitlab, to avoid API shortcomings.

You need to first create a ssh key, then add the public part to Gitlab (see this [guide](https://docs.gitlab.com/ee/ssh/))

Then, you need to create the secret to add the ssh key, and the following config to your container

```
host gitlab.com
  HostName gitlab.com
  StrictHostKeyChecking no
  IdentityFile ~/.ssh/id_rsa
  User git
```

To easily create the secret, you can do the following (see [docs](https://kubernetes.io/docs/concepts/configuration/secret/#use-case-pod-with-ssh-keys))

```sh
kubectl create secret generic ssh-key-secret --from-file=config=/path/to/config --from-file=id_rsa=/path/to/.ssh/id_rsa --from-file=id_rsa.pub=/path/to/.ssh/id_rsa.pub
```

It creates something like this

```yml
apiVersion: v1
data:
  config: aG9zdCBnaXRsYWIuY29tCiAgSG9zdE5hbWUgZ2l0bGFiLmNvbQogIFN0cmljdEhvc3RLZXlDaGVja2luZyBubwogIElkZW50aXR5RmlsZSB+Ly5zc2gvaWRfcnNhCiAgVXNlciBnaXQ=
  id_rsa: <base64String>
  id_rsa.pub: <base64String>
kind: Secret
metadata:
  name: ssh-key-secret
  namespace: <namespace>
```

Then you just need to add Git author, and mount volumes
The final configuration should look like something like this :

```yml
---
apiVersion: v1
kind: Namespace
metadata:
  name: <namespace, for example renovate>

---
apiVersion: v1
kind: Secret
metadata:
  name: renovate-env
  namespace: <namespace>
type: Opaque
stringData:
  GITHUB_COM_TOKEN: 'any-personal-user-token-for-github-com-for-fetching-changelogs'
  RENOVATE_AUTODISCOVER: 'false'
  RENOVATE_ENDPOINT: 'https://github.company.com/api/v3'
  RENOVATE_GIT_AUTHOR: 'Renovate Bot <bot@renovateapp.com>'
  RENOVATE_PLATFORM: 'github'
  RENOVATE_TOKEN: 'your-github-enterprise-renovate-user-token'
---
apiVersion: v1
data:
  config: aG9zdCBnaXRsYWIuY29tCiAgSG9zdE5hbWUgZ2l0bGFiLmNvbQogIFN0cmljdEhvc3RLZXlDaGVja2luZyBubwogIElkZW50aXR5RmlsZSB+Ly5zc2gvaWRfcnNhCiAgVXNlciBnaXQ=
  id_rsa: <base64String>
  id_rsa.pub: <base64String>
kind: Secret
metadata:
  name: ssh-key-secret
  namespace: <namespace>
---
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: renovate
  namespace: <namespace>
spec:
  schedule: '@hourly'
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          volumes:
            - name: ssh-key-volume
              secret:
                secretName: ssh-key-secret
          containers:
            - name: renovate
              # Update this to the latest available and then enable Renovate on the manifest
              image: renovate/renovate:23.19.2
              volumeMounts:
                - name: ssh-key-volume
                  readOnly: true
                  mountPath: '/home/ubuntu/.ssh'
              args:
                - <repository>
              # Environment Variables
              envFrom:
                - secretRef:
                    name: renovate-env
          restartPolicy: Never
```

## Logging

It's recommended to configure `LOG_LEVEL=debug` and `LOG_FORMAT=json` in environment if you are ingesting/parsing logs into another system. Debug logging is usually necessary for any debugging, while JSON format will mean that the output is parseable.
