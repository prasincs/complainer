# Complainer

Complainer's job is to send notifications to different services when tasks
fail on Mesos cluster. While your system should be reliable to failures of
individual tasks, it's nice to know when things fail and why.

Supported log upload services:

* No-op - keeps URLs to Mesos slave sandbox.
* S3 - both AWS S3 and on-premise S3-compatible API.

Supported reporting services:

* [Sentry](https://getsentry.com/) - a great crash reporting software.
* [Hipchat](https://www.hipchat.com/) - not so great communication platform.
* [Slack](https://slack.com/) - another communication platform.

## Quick start

Start sending all failures to Sentry:

```
docker run -it --rm cloudflare/complainer \
  -masters=http://mesos.master:5050 \
  -uploader=noop \
  -reporters=sentry \
  -sentry.dsn=https://foo:bar@sentry.dsn.here/8
```

Run this on Mesos itself!

![Sentry screenshot](screenshots/sentry.png)

## Reporting configuration

Complainer needs two command line flags to configure itself:

* `name` - Complainer instance name (default is `default`).
* `masters` - Mesos master URL list (ex: `http://host:port,http://host:port`).

### Log upload services

Log upload service is specified by command line flag `uploader`. Only one
uploader can be specified per complainer instance.

#### no-op

Uploader name: `noop`

No-op uploader just echoes Mesos slave sandbox URLs.

#### S3

Uploader name: `s3`.

Stdout and stderr logs get uploaded to S3 and signed URLs provided to reporters.
Logs are uploaded into the following directory structure:

* `complainer/${task_name}/${YYYY-MM-DDTHH:mm:ssZ}-${task_id}/{stdout,stderr}`

Command line flags:

* `s3.access_key` - S3 access key.
* `s3.secret_key` - S3 secret key.
* `s3.endpoint` - S3 endpoint (ex: `https://s3-ap-southeast-1.amazonaws.com`).
* `s3.bucket_endpoint` - S3 bucket endpoint, might be needed for S3-compatible
APIs (ex: `https://${bucket}.my.custom.domain`).
* `s3.bucket` - S3 bucket name.
* `s3.timeout` - Timeout for signed S3 URLs (ex: `72h`).

You can set value of any command line flag via environment variable, just
replace dots with underscores and capitalise letters. Example:

* Flag `s3.access_key` becomes env variable `S3_ACCESS_KEY`

Flags override env variables if both are supplied.

### Reporting services

Reporting services are specified by command line flag `reporters`. Several
services can be specified, separated by comma.

#### Sentry

Command line flags:

* `sentry.dsn` - Default Sentry DSN to use for reporting.

Labels:

* `dsn` - Sentry DSN to use for reporting.

If label is unspecified, command line flag value is used.

#### Hipchat

Command line flags:

* `hipchat.base_url` - Base Hipchat URL, needed for on-premise installations.
* `hipchat.room` - Default Hipchat room ID to send notifications to.
* `hipchat.token` - Default Hipchat token to authorize requests.

Labels:

* `base_url` - Hipchat URL, needed for on-premise installations.
* `room` - Hipchat room ID to send notifications to.
* `token` - Hipchat token to authorize requests.

If label is unspecified, command line flag value is used.

#### Slack

Command line flags:

* `slack.hook_url` - Webhook URL, needed to post something (required).
* `slack.channel` - Channel to post into, e.g. #mesos (optional).
* `slack.username` - Username to post with, e.g. "Mesos Cluster" (optional).
* `slack.icon_emoji` - Icon Emoji to post with, e.g. ":mesos:" (optional).
* `slack.icon_url` - Icon URL to post with, e.g. "http://my.com/pic.png" (optional).

Labels:

* `hook_url` - Webhook URL, needed to post something (required).
* `channel` - Channel to post into, e.g. #mesos (optional).
* `username` - Username to post with, e.g. "Mesos Cluster" (optional).
* `icon_emoji` - Icon Emoji to post with, e.g. ":mesos:" (optional).
* `icon_url` - Icon URL to post with, e.g. "http://my.com/avatar.png" (optional).

If label is unspecified, command line flag value is used.

For more details see [Slack API docs](https://api.slack.com/incoming-webhooks).

### Label configuration

#### Basics

To support flexible notification system, Mesos task labels are used. Marathon
task labels get copied to Mesos labels, so these are equivalent.

The minimal set of labels needed is an empty set. You can configure default
values in Complainer's command line flags and get all notifications with
these settings. In practice, you might want to have different reporters for
different apps.

Full format for complainer label name looks like this:

* `complainer_${name}_${reporter}_instance_${instance}_${key}`

Example (`dsn` set for `default` Sentry of `default` Complainer):

* `complainer_default_sentry_instance_default_dsn`

This is long and complex, so default parts can be skipped:

* `complainer_sentry_dsn`

#### Advanced labels

The reason for having long label name version is to add the flexibility.
Imagine you want to report app failures to the internal Sentry, two internal
Hipchat rooms (default and project-specific) and the external Sentry.

Set of labels would look like this:

* `complainer_sentry_dsn: ABC` - for internal Sentry.
* `complainer_hipchat_instances: default,myapp` - adding instance `myapp`.
* `complainer_hipchat_instance_myapp_room: 123`- setting room for `myapp`.
* `complainer_hipchat_instance_myapp_token: XYZ`- setting token for `myapp`.
* `complainer_external_sentry_dsn: FOO` - for external Sentry.

Internal and external complainers can have different upload services.

#### Dogfooding

To report errors for complainer itself you need to run two instances:

* `default` to monitor all other tasks.
* `dogfood` to monitor the `default` Complainer.

You'll need the following labels for the `default` instance:

```yaml
labels:
  complainer_dogfood_sentry_dsn: sentry-dsn-here
  complainer_dogfood_hipchat_room: 'hipchat-room-id-here'
```

For the `dogfood` instance you'll need to:

* Add `-name=dogfood` command line flag.
* Skip `-sentry.dsn` command line flag.
* Skip `-hipchat.room` command line flags.

Since Complainer ignores reporters with incomplete configurations, the `dogfood`
instance would ignore every failure except for the `default` instance failures.

If the `dogfood` instance fails, `default` reports it just like any other task.

If both instances fail at the same time, you get nothing.

## Copyright

* Copyright 2016 CloudFlare

## License

MIT
