# AltPostLocal

Replacement for Postfix's local(8) and virtual(8).

# Configuration

## File

Configuration file is `/etc/altpostlocal/altpostlocal.yaml`.

This is a [YAML 1.1](https://yaml.org/spec/1.1/) file.

Please remind anchors and aliases because these are very useful.

## Root params

|Key|Type|Description|
|------|------|---------------------|
|`extensions`|`String`|Optional. See Extensions section.|
|`rematch`|`Mapping`|Optional. See REMatch section.|
|`aliases`|`Mapping`|See Aliases section.|
|`default`|`Action[]`|Delivery action when alias matching failed.|
|`rescue`|`String`|Optional. Writable directory path. Save email to there when action returns error. If this setting does not exist, exit status 78 (EX_CONFIG = configuration error) is returned.|
|`rescue_json`|`Boolean`|Optional. If ture, save email as JSON with `dump` format on fail.|
|`die_quietly`|`Boolean`|Optional. If true, always exit 0 even if lost email.|

## Extensions

`Extensions` value is a Onigmo's RegExp string.

When alias matching, Aliases function finds longest address key.
For example, destination address is `foo+bar+baz@example.com` and `Extensions` is `+`; Aliases function see `foo+bar+baz@example.com` first, `foo+bar@example.com` next, and `foo@exmaple.com` last.

`Extensions` value is a *RegExp* string, the behavior of Extensions is shown in simple code as follows:

```ruby
extensions = Regexp.new(config["extensions"])
username.rindex(extensions)
```

## REMatch

`REMatch` is a mapping.
Key is a Onigmo's RegExp string, value is a replacement address.

REMatch is performed before normal matching and replaces the recipient address.

Caution: REMatch can potentially slow down script processing significantly.

The result of REMatch also affects the value of `${recipient}` in `pipe`.

## Aliases

Aliases is a `Mapping`.

```
address: Action[]
```

`address` is email address *includes domain* like virtual(5). If domain is not included, matching is done by user name alone, regardless of domain.

For example, `foo@example.com` matches `foo@exmaple.com` or with extension like `foo+bar@example.com`, and `foo` matches `foo@example.com`, `foo@example.net` or `foo`.

Aliases are matched using the `String#downcase` method and **must be lowercase**.

If an alias containing a domain is not matched, the address is matched excluding the domain.
For this reason, addresses such as abuse and mailer-daemon, which need to capture mail but do not need to be configured by domain, can be written easily by writing only aliases that do not include a domain.

Special alias domain `@.` matches just user.
For exmaple, `jrh@.` matches *just only* `jrh`.

If you want alias in aliases, please use YAML's anchor and aliases.

## Action

### Maildir

```yaml
type: maildir
value: <dir>
```

Save on `dir` as a Maildir.
`dir` should be absolute directory path.

### pipe

```yaml
type: pipe
cmd: <command>,
args: [<arguments...>]
timeout: <int>
onerror: Action
```

Write pipe to `cmd` with `args`.

`pipe` expands `${sender}` or `${receipient}` on `args`.

If optional `timeout` is set, raise error after `int` seconds.

If optional `onerror` os set, treat specified action when pipe raises exception or pipe returns non-zero status instead of global rescue action.

Hint: `args` is useful for override alias.

### Dump

```yaml
type: dump
value: <dir>
```

Save on `dir` with JSON format.

JSON data is

```json
{
  "sender": "sender_address",
  "recipients": "recipient_addresses",
  "mail": "mail_body"
}
```

Dumped JSON can be attempted to be redistributed using `altpostlocal -j jsonfile.json`.

### Nothing

```yaml
type: nothing
```

Do nothing.

This is useful if you simply want to destroy an e-mail.

### NoUser

```yaml
type: nouser
```

Exit with status 67 immidiately.

Caution: *This action ends script process.*

### forward

```yaml
type: forward
value: [<address...>]
```

Forward to address(es) with `sendmail` command.

```yaml
type: forward
value: [foo@example.com]
```

is same as

```yaml
type: pipe
cmd: sendmail
args: [foo@example.com]
```

# Replace Postfix's local/virtual

## configure master.cf

Add altpostlocal to `master.cf`.

```
altpostlocal    unix  -       n       n       -       -       pipe
  flags=F user=email argv=/usr/local/bin/altpostlocal -f ${sender} -- ${recipient}
```

user and argv should be change according to your environment.

## configure main.cf

### local\_transport

Set altpostlocal to `local_transport`.

```
local_transport = altpostlocal
```

### local\_recipient\_maps

Set empty to `local_recipient_maps` for never reject e-mail to the destination listed in mydestination.

```
local_recipient_maps =
```

### mydestination

List *all* destination domains in `mydestination`.

```
mydestination = example.com, example.net, example.org, example.info
```

Note: Don't use `virtual_mailbox_domains`.


