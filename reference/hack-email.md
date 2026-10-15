# E-mail Sending

`$core->email` (class `eMailSender`) sends e-mail. It is loaded lazily
on first access — see [The Core](hack-core.md). Most hacks rarely need it; the platform
itself uses it for registration, recovery and notifications.

It exposes a single public method:

```php
$core->email->send( $to, $title, $text, $files = [] );
```

| Argument | Meaning |
|----------|---------|
| `$to` | One address (string) or an array of addresses (sent individually). |
| `$title` | Subject line — UTF-8 base64-encoded for you. |
| `$text` | Body. If it contains a `<`, it is sent as `text/html`, otherwise as plain text. |
| `$files` | Attachments as `[ '/abs/path' => 'display-name.ext', … ]`. |

```php
$core->email->send(
	$core->user->email,
	$core->lang['xproxy_ready_subj'],
	$core->lang['xproxy_ready_body'],          // contains <p>…</p> → sent as HTML
	[ PATH_DATA . $report => 'report.pdf' ]    // optional attachment
);
```

`send()` is fire-and-forget — it does not return a useful delivery status, so do not
rely on its return value.

## Configuration

Transport is chosen from the `mail` config block: if `smtp` is set the message goes over
SMTP, otherwise through PHP's `mail()`. The block fields:

| `config( 'mail', … )` | Purpose |
|-----------------------|---------|
| `from` | Envelope / `From:` address. |
| `smtp` | Truthy to use SMTP instead of `mail()`. |
| `server` | SMTP server as `proto:host` (`smtp:`, `smtps:`, `ssl:`, `tls:`). |
| `port` | SMTP port. |
| `user` / `pass` | SMTP credentials. |
| `domain` | HELO/EHLO domain (optional). |

These are managed in the control panel's mail settings; a hack normally just calls
`send()` and lets the site's configuration decide the transport. See
[Configuration](hack-core.md#configuration) for how config blocks are read.
