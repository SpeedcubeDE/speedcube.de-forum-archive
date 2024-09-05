The Speedcube.de forum used to be a German Speedcubing forum based on the [MyBB](https://mybb.com/) forum software.  
It was archived in September of 2024 and replaced by a static read-only version.

Archiving was done by running the following `wget` command against the forum:
```bash
wget \
  --mirror \
  --page-requisites \
  --convert-links \
  --adjust-extension \
  --wait 0 \
  --restrict-file-names=windows \
  --span-hosts \
  --domains=forum.speedcube.de,speedcubers.de \
  --trust-server-names \
  https://forum.speedcube.de/
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
        root /path/to/forumbackup/forum.speedcube.de;
        index index.html index.php.html;
    }

    # HTTP/80 works, but in reality it's HTTPS/443, auto-configured using letsencrypt's certbot
    listen 80;
}
```

Additionally, `speedcubers.de` used to be a redirect to `forum.speedcubers.de`. For that, see [github.com/SpeedcubeDE/speedcubers-redirect](https://github.com/SpeedcubeDE/speedcubers-redirect).
