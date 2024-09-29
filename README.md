The Speedcube.de forum used to be a German Speedcubing forum based on the [MyBB](https://mybb.com/) forum software.  
It was archived in September of 2024 and replaced by a static read-only version.

Archiving was done by running the following `wget` command against the forum:
```bash
wget \
  --recursive \
  --level inf \
  --page-requisites \
  --convert-links \
  --adjust-extension \
  --restrict-file-names=windows \
  --wait 0 \
  --span-hosts --domains=forum.speedcube.de,speedcubers.de \
  --trust-server-names \
  https://forum.speedcube.de/
```

The above command was chosen based on these considerations:
- Don't use `--mirror` and instead specify `--resursive` and `--level inf` separately, because `--mirror` also implies `--timestamping`,
  and timestamping was not supported by the server anyway.
- Use `--page-requisites`, `--convert-links` and `--adjust-extension` to make the archive completely self-contained and statically servable.
- Use `--restrict-file-names=windows` to make the converted filenames more portable. Notably to replace the query parameters' `?` with `@`.
- Use `-wait 0` to speed up the process, since the old server did not impose any rate limits anyway.
- Use `--span-hosts --domains=forum.speedcube.de,speedcubers.de` to also traverse `speedcubers.de`, which just redirects back to `forum.speedcube.de`.
- Use `--trust-server-names` to have the `speedcubers.de` redirects render as their redirect target in the produced archive,
  so the redundant redirect hops through `speedcubers.de` become regular relative links in the final result.
- The `--no-clobber` option would have been used to be able to resume partial downloads by having wget respect and use files already existing on disk.
  However, this does not work as `wget` claims the option is incompatible with `--convert-links`.
  I was unable to get `wget` to resume any partial downloads and had to start over each time my hard drive got full.
  See also [https://savannah.gnu.org/bugs/index.php?62782#comment2](https://savannah.gnu.org/bugs/index.php?62782#comment2) for a comment of mine on the gnu issue tracker.

The command took 1 week to traverse the entire forum and download all files, and 1 more week to convert all links.
The majority of the slowdown was caused by wget having to parse posts in threaded mode,
which are not only many in number, but also each very large. See for example [Thread #7728](https://forum.speedcube.de/showthread.php?tid=7728&pid=197793&mode=threaded).

Because the produced mirror is around 260GB large, and buying a server with that much disk space just for serving it would be expensive,
the final mirror was compressed into a [SquashFS](https://docs.kernel.org/filesystems/squashfs.html) file instead:

```sh
mksquashfs forum.speedcube.de forum_speedcube_de_archive.sqsh -comp zstd
```

SquashFS not only supports good compression (I used zstd in this case) but also efficient random access.
The resulting file only measures ~13GB can then be mounted, allowing access to the individual files again.
This requires ad-hoc decompression of each accessed file, which is well worth it.

```sh
mkdir /mnt/forum_speedcube_de_archive
mount -t squashfs -o loop forum_speedcube_de_archive.sqsh /mnt/forum_speedcube_de_archive
```

The produced static files are now served via a nginx config like this:
```nginx
server {
    server_name forum.speedcube.de;

    # rewrite does not work with query params
    # so we're using ifs and returns instead.

    # Some Actions (e.g. action=lastpost) used to dynamically determine where to navigate,
    # and then navigate to an actual canonical URL, e.g. to the respective page of the newest post.
    # They don't make sense anymore since all content is static now, so remove them.
    if ($request_uri ~ ^(.*?[?&])action=(?:lastpost)&(.*)?$) {
        return 301 $1$2;
    }
    if ($request_uri ~ ^(.*?)[?&]action=(?:lastpost)$) {
        return 301 $1;
    }
    # The query parameter separator '?' was replaced by '@',
    # and all files got '.html' appended.
    # Some files, e.g. under /archive, might already have an '.html' extension.
    if ($request_uri ~ ^(.*?)\.php\?(.*?)(?:\.html)?$) {
        return 301 $1.php@$2.html;
    }
    # redirect index.php to index.php.html
    if ($request_uri ~ ^(.*?)\.php$) {
        return 301 $1.php.html;
    }

    location / {
        root /mnt/forum_speedcube_de_archive;
        index index.html index.php.html;
    }

    # HTTP/80 works, but in reality it's HTTPS/443, auto-configured using letsencrypt's certbot
    listen 80;
}
```

Additionally, `speedcubers.de` used to be a redirect to `forum.speedcubers.de`. For that, see [github.com/SpeedcubeDE/speedcubers-redirect](https://github.com/SpeedcubeDE/speedcubers-redirect).
