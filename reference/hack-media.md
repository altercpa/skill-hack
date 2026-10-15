# Media Library

`$core->media` (class `MediaLib`) handles uploaded and generated files
— images, documents, archives — with extension whitelisting, deduplication, image
conversion and resizing. It is loaded lazily on first access — see
[The Core](hack-core.md).

Everything is organised into named **libraries**, each with its own allowed extensions
and storage layout. Files physically live under `data/<lib>/…`; the methods return a
path **relative to `data/`** (e.g. `pics/ab/cd/….jpg`), which is what you store in the
database. To render it, prefix `data/` and pass it through `$core->u()` / `$core->uri()`.

## Built-in Libraries

| Library | Allowed | Storage `mode` | Notes |
|---------|---------|----------------|-------|
| `news` | images | generic | Overwrites by name |
| `ads` | (global list) | generic | Overwrites by name |
| `avatar` | images | generic, `ext=jpg` | Converts to JPEG |
| `offer` | images | generic, `ext=jpg` | Converts to JPEG |
| `audio` | mp3/wav/ogg | `mdf` (content hash) | Deduplicated |
| `pics` | images | `mdf` (content hash) | Deduplicated; target of `text()` |
| `bonus` | images | generic | |
| `lead` | (global list) | `mdf` (content hash) | Deduplicated |
| `sites` | zip | generic, `ext=zip` | |

When a library has no `allow` list, the global whitelist applies (images, video, audio,
`pdf`, office docs, archives, `txt`/`rtf`/`fb2`/`epub`, …).

### Library config keys

A library definition understands:

| Key | Meaning |
|-----|---------|
| `allow` | Array of allowed extensions (overrides the global whitelist). |
| `ext` | Force this extension; the original name is ignored. |
| `overwrite` | `true` overwrites a same-named file; `false` appends `-1`, `-2`, … |
| `mode` | Storage layout — see below. |
| `convert` | Convert images to this format (e.g. `jpeg`). |

### Storage modes

| Mode | Layout | Use |
|------|--------|-----|
| *(default)* | `data/<lib>/<slug>.<ext>` | Human-readable names. |
| `md5` | `data/<lib>/ab/cd/<rest>.<ext>` (hash of the name) | Spread many files across folders. |
| `mdf` | same nested layout, hash of the **content** | Deduplicate — identical files stored once. |
| `numeric` | `data/<lib>/<n÷1000>/<n>.<ext>` | Numerically-keyed entities. |

## Methods

| Method | Purpose |
|--------|---------|
| `upload( $lib, $file, $name = false )` | Store a `$_FILES` entry in a library. Validates `is_uploaded_file` and the extension. Returns the `data/`-relative path, or `false`. |
| `move( $lib, $name, $file )` | Store an existing file path in a library (handles conversion). Returns the path or `false`. |
| `path( $lib, $name )` | Compute the public path (`data/<lib>/…`) for a stored name. In default mode it uses `$name` verbatim (unlike `move()`, which slugifies via `text->link()`), so store and reuse the exact path `move()`/`upload()` returned rather than recomputing it. |
| `text( $html )` | Extract inline `data:image/…;base64,…` images, save them to `pics`, and replace them with absolute URLs. Returns the rewritten HTML. |
| `resize( $src, $dst, $minx, $miny, $cut = false, $show = false )` | Resize / crop an image. |
| `crop( $src, $dst, $h, $w )` | Shortcut for `resize( …, $cut = 1 )`. |
| `name( $n )` / `ext( $n )` | Filename without extension / lowercase extension. |

### Uploading

```php
// A form file field <input type="file" name="pic">
$path = $core->media->upload( 'pics', $core->files['pic'] );
if ( ! $path ) $core->msgo( 'badfile', $back );

// Store the relative path; render it later
$core->db->edit( DB_XPROXY_ADDR, [ 'proxy_pic' => $path ], [ 'proxy_id' => $id ] );
$url = $core->u( 'data/' . $path );      // /data/pics/ab/cd/….jpg
```

`upload()` returns `false` on a missing file, a non-upload, an unknown library, or a
disallowed extension — always check it.

### Registering your own library

`MediaLib` fires the `medialibs` filter when it is constructed — but **only if at least
one handler is registered** (it is guarded by `$core->cando( 'medialibs' )`).
Subscribing in your hack's `init` satisfies that guard automatically. Because `MediaLib`
is lazy-loaded on first `$core->media` access, register the handler in `init` (which runs
before any media access) so your library is in place by the time `upload()` is called:

```php
// In hack_proxy_init():
$core->handle( 'medialibs', 'hack_proxy_medialibs' );

function hack_proxy_medialibs( $core, $libs ) {
	$libs['xproxy'] = [
		'allow'		=> [ 'jpg', 'jpeg', 'png', 'webp' ],
		'mode'		=> 'mdf',          // dedupe by content
		'overwrite'	=> true,
		'convert'	=> 'jpeg',
	];
	return $libs;
}
```

After this, `$core->media->upload( 'xproxy', … )` stores into `data/xproxy/…`.

### Inline images from editor content

`text()` turns pasted base64 images into real files — this is what
`$core->text->mcode()` calls, so WYSIWYG content gets normalised automatically. Use it
directly when you process HTML yourself:

```php
$html = $core->media->text( $core->text->tcode( $core->post['body'] ?? '' ) );
```

### Resizing and thumbnails

```php
// Write a 200×200 cropped thumbnail to a file
$core->media->crop( PATH_DATA . $path, PATH_DATA . 'pics/thumb/' . basename( $path ), 200, 200 );

// Fit within 800×600, keep aspect ratio
$core->media->resize( $src, $dst, 800, 600 );

// Stream a resized image straight to the browser (no $dst)
$core->media->resize( $src, false, 400, 0 );   // width 400, height auto
```

`$dst` may be a file path, a numeric JPEG quality (then the image is sent to the
browser), or `false` (browser output at quality 90). `$cut`: `false` fits inside the
box, `1` crops to fill, `2` does a square-fit cut. Pass `$show = true` to also send the
written file as a download.
