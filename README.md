# AltPostLocal

Replacement for Postfix's local(8) and virtual(8).

# Configuration

## File

Configuration file is `/etc/altpostlocal/altpostlocal.yaml`.

This is a [YAML 1.1](https://yaml.org/spec/1.1/) file.

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
extensions = RegExp.new(config["extensions"])
username.rindex(extensions)
```

## REMatch

`REMatch` is a mapping.
Key is a Onigmo's RegExp string, value is a replacement address.

REMatch is performed before normal matching and replaces the recipient address.

Caution: REMatch can potentially slow down script processing significantly.

## Aliases

Aliases is a `Mapping`.

```
address: Action[]
```

`address` is email address *includes domain* like virtual(5). If domain is not included, matching is done by user name alone, regardless of domain.

For example, `foo@example.com` matches `foo@exmaple.com` or with extension like `foo+bar@example.com`, and `foo` matches `foo@example.com`, `foo@example.net` or `foo`.

## Action

### Alias

```yaml
type: alias
value: <address>
```

Replace current address to `value` and re-matching.

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
value: [<command>, <arguments...>]
```

Write pipe to `cmd` with `args`.

`pipe` expands `${sender}` or `${receipient}` on `args`.

### dump

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

# Replace Postfix's local/virtual

## configure master.cf

Add altpostlocal to `master.cf`.

```
altpostlocal    unix  -       n       n       -       -       pipe
  flags=F user=email argv=/usr/local/bin/altpostlocal -f ${sender} -- ${recipient}
```

user and argv should be rewritten according to your environment.

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


