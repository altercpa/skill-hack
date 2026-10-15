# Text Processing

`$core->text` (class `texter`) is the toolbox for cleaning incoming
text and shaping outgoing text. It is loaded lazily on first access — see
[The Core](hack-core.md).

The rule of thumb: **sanitize on input, render on output.** Sanitizers return a cleaned
value; validators return the value or `false`.

## Sanitize / Validate Input

| Method | Purpose |
|--------|---------|
| `line( $t )` | Plain single-line text (tags stripped, trimmed). |
| `anum( $t )` | Alphanumeric line (word chars and `_-.,` only). |
| `link( $t )` | URL-safe slug (transliterates Cyrillic, lowercases). |
| `email( $t )` | Valid e-mail, or `false`. |
| `url( $t )` | Valid URL, or `false`. |
| `float( $n )` | Float (accepts a comma decimal), or `false`. |
| `decimal( $n, $round = false )` | Integer when whole, otherwise a float. |
| `number( $n )` | Digits only. |
| `domain( $u )` | Bare host (drops scheme / path / query / `www.`). |
| `tags( $t )` | Normalised, de-duplicated space-separated tag line. |
| `ints( $t )` / `codes( $t )` | Array of unique ints / slugs from a CSV string. `ints()` drops zeros. |
| `intz( $t )` | Like `ints()` but **keeps zero values**. |
| `intline( $t, $zeros = false )` / `codeline( $t )` | Same, returned as a cleaned CSV string. `$zeros = true` keeps zeros. |
| `cols( $t, $nol = false )` | Array of slugs (or lines) split on whitespace/commas. |
| `pass( $t, $sid = 0 )` | Hash a password. |
| `ass( $t )` | Clean and re-escape a string for safe SQL re-insertion. |

```php
$slug  = $core->text->link( $core->post['title'] ?? '' );
$email = $core->text->email( $core->post['mail'] ?? '' );
if ( ! $email ) $core->msgo( 'bademail', $back );

$ids = $core->text->ints( $core->post['ids'] ?? '' );   // [12, 34, 56]
```

> `ass()` is the safe way to put a value **read back from the database** into another
> query — DB reads are not auto-escaped (see [SQL safety](hack-db.md#sql-safety)).

## Render / Decode Output

| Method | Purpose |
|--------|---------|
| `html( $t )` | Escape for HTML output (`htmlspecialchars`). |
| `untag( $t )` | Strip tags + decode entities + unslash. |
| `unspec( $t )` | Decode entities + unslash (keeps tags). |
| `unquote( $t )` | Escape `"` for attribute values. |
| `metasafe( $t )` | Tag-free, quote-safe `<meta>` content. |
| `xml( $t )` | Escape `< > &` for XML. |
| `code( $t )` / `tcode( $t )` / `mcode( $t )` | Pass HTML through filters (with editor cleanup / media expansion). |
| `lines( $t )` / `links( $t )` | Paragraph/`<br>` formatting / auto-link URLs and e-mails. |
| `unurl( $t )` | Rewrite outgoing `<a href>` links through the `/?goto=` redirector with `rel="nofollow"`. |
| `excerpt( $t, $len = 100 )` / `cut( $t, $len = 35 )` | Truncate to a word boundary / a hard length. |
| `mask( $t )` / `emask( $t )` | Mask a string / an e-mail for display. |

```php
echo $core->text->html( $name );           // safe in HTML body
echo $core->text->cut( $title, 40 );        // "Long title that is cut he…"
echo $core->text->emask( $user_email );     // "jo***@gmail.com"
```

## Time Helpers

| Method | Purpose |
|--------|---------|
| `smartdate( $tm )` | "Today/yesterday HH:MM", else `d.m.y H:i`. |
| `timeleft( $to, $from = false )` | Human "N days / hours / minutes left". |
| `timedelta( $to, $from = false )` | Compact signed delta (`+3h`, `-12m`). |
| `search( $t )` | Tokenised, normalised search string. |

```php
$when = $core->text->smartdate( $row['proxy_checked'] );
$left = $core->text->timeleft( $row['task_deadline'] );
```

These read plural/relative wordings from `$core->lang`, so they respect the active
language; see [Language files](hack-lang.md).
