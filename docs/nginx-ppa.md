# Building nginx-wo in ppa:emanweb-wordops/nginx-wo

Runbook for publishing patched `nginx-wo` builds when the upstream
`ppa:wordops/nginx-wo` lags behind (WordOps issue 785).

Target release: nginx **1.30.6** (14 Oct 2026). It fixes the 1.30.4 regression
that segfaults workers in third-party modules using script codes
(`set_escape_uri` in `--wpredis` sites; nginx PR #1601), plus CVE-2026-15176,
CVE-2026-93479, CVE-2026-95101, CVE-2026-97629 and CVE-2026-97205.

Package version scheme: `1.30.6-1ppa~emanweb~bpo24.04.1` (noble),
`1.30.6-1ppa~emanweb~bpo22.04.1` (jammy). These sort above every upstream
1.30.4/1.30.5 build.

## 1. One-time setup

1. Create the PPA: <https://launchpad.net/~emanweb-wordops/+activate-ppa>,
   name `nginx-wo`.
2. In the PPA's **Edit PPA dependencies** page, add `ppa:wordops/nginx-wo`.
   The upstream package build-depends on openssl, pcre3, brotli and debhelper
   builds that only exist in that PPA.
3. On a build machine (Ubuntu 24.04, not a production server):

   ```bash
   sudo apt install devscripts dput ubuntu-dev-tools quilt gnupg
   export DEBFULLNAME="Emanuel Costa" DEBEMAIL="emanweb@gmail.com"
   gpg --list-secret-keys --keyid-format long   # key must be on Launchpad
   ```

## 2. Build the source package

```bash
mkdir -p ~/nginx-build && cd ~/nginx-build

# upstream WordOps packaging (currently 1.30.5)
pull-ppa-source --ppa wordops/nginx-wo nginx noble

# nginx 1.30.6 release, signature-verified
wget https://nginx.org/download/nginx-1.30.6.tar.gz{,.asc}
gpg --verify nginx-1.30.6.tar.gz.asc   # import keys from nginx.org/en/pgp_keys.html first

cd nginx-1.30.5
uupdate -u ../nginx-1.30.6.tar.gz -v 1.30.6
cd ../nginx-1.30.6
```

Check the patch series before going further:

```bash
export QUILT_PATCHES=debian/patches
quilt push -a && quilt pop -a
```

- Drop any patch that backports nginx PR #1601 (from the 1.30.5-2 OBS request);
  it is already in 1.30.6.
- If a third-party module bundled under `debian/` fails to build against
  1.30.6, update that module before continuing.

Set the version and build the signed source upload:

```bash
dch -v 1.30.6-1ppa~emanweb~bpo24.04.1 -D noble \
  "New upstream release 1.30.6. Fixes worker segfaults with set-misc (WordOps issue 785) and five CVEs."
debuild -S -sa -k<GPG_KEY_ID>
```

Optional local binary test build before uploading: `debuild -b -uc -us`.

## 3. Upload

```bash
cd ~/nginx-build
dput ppa:emanweb-wordops/nginx-wo nginx_1.30.6-1ppa~emanweb~bpo24.04.1_source.changes
```

Launchpad emails the accept/reject result, then builds amd64 (and any other
enabled architectures). For jammy, repeat step 2 from the jammy source with
`-D jammy` and the `bpo22.04.1` version.

## 4. Test on staging

1. Reproduce first: a `--wpredis` site on nginx 1.30.4/1.30.5 should log
   `exited on signal 11` in `/var/log/nginx/error.log`.
2. Install the new build (section 5), then:

   ```bash
   nginx -v                                   # 1.30.6
   for i in $(seq 200); do curl -s -o /dev/null https://staging.example.com/; done
   journalctl -k --since "10 min ago" | grep -i segfault   # expect nothing
   grep "signal 11" /var/log/nginx/error.log               # expect nothing new
   ```

## 5. Switch a server to this PPA

Keep the upstream PPA enabled (it carries runtime dependencies), and pin the
nginx packages to ours:

```bash
sudo add-apt-repository -y ppa:emanweb-wordops/nginx-wo
sudo tee /etc/apt/preferences.d/nginx-wo-emanweb >/dev/null <<'EOF'
Package: nginx-wo nginx-custom
Pin: release o=LP-PPA-emanweb-wordops-nginx-wo
Pin-Priority: 1001
EOF
sudo apt update && apt-cache policy nginx-wo     # candidate should be 1.30.6-1ppa~emanweb~...
sudo apt-mark unhold nginx-wo nginx-custom
sudo apt install nginx-wo nginx-custom
sudo nginx -t && sudo systemctl restart nginx
```

For new installs, set this in `/etc/wo/wo.conf` before `wo stack install`:

```ini
[nginx]
ppa = ppa:emanweb-wordops/nginx-wo
```

To return to upstream later, delete `/etc/apt/preferences.d/nginx-wo-emanweb`,
remove our PPA and reinstall `nginx-wo` from `ppa:wordops/nginx-wo`.
