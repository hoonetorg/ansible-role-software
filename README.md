# ansible-role-software

Installs the packages listed in the inventory (`state: present`, no automatic upgrades) and, where defined,
language packs.

1. `software` from the inventory is merged over `software_defaults` into `software_final`
2. OS vars: `vars/<distribution>.yml`, else `vars/<os_family>.yml`, else none (`vars/empty.yml`)
3. language packs: only if the OS vars define `software_languagepkgs_map` (Debian, Ubuntu, RedHat/Fedora);
   openSUSE: none yet (English/system default only)
4. vendor packages (Microsoft Edge, Google Chrome, Brave): their empty defaults file (`/etc/default/microsoft-edge`, …)
   is created before the installation if it does not exist - otherwise the package's install script adds its own
   repo file next to the one from ansible-role-repo (the same trick `opi msedge` uses)
5. `software.packages` plus `software.packages_by_os[<os_family>]` are installed; an entry
   `{name: <package>, from: <repo alias>}` is installed from exactly that repository (openSUSE only, see below),
   before the plain names
6. browser policies, only for browsers in the package list (otherwise the role's own policy file is removed):

   | browser | package (match) | policy file | check |
   |---|---|---|---|
   | Firefox | `MozillaFirefox`, `firefox`, `firefox-esr` | `/etc/firefox/policies/policies.json` | `about:policies` |
   | Chromium | `chromium`, `chromium-browser` | `/etc/chromium/policies/managed/ansible.json` | `chrome://policy` |
   | Chrome | `google-chrome-*` | `/etc/opt/chrome/policies/managed/ansible.json` | `chrome://policy` |
   | Brave | `brave-browser*` | `/etc/brave/policies/managed/ansible.json` | `brave://policy` |
   | Edge | `microsoft-edge-*` | `/etc/opt/edge/policies/managed/ansible.json` | `edge://policy` |

   Chromium based browsers: `chromium_common_policies`, plus `ExtensionSettings` for the extensions of their store
   (`chrome_store_extensions` for Chromium/Chrome/Brave, `edge_store_extensions` for Edge; `normal_installed`),
   plus `<browser>_policies` (merged last, the browser's value wins). Firefox: `{"policies": firefox_policies}`.
   Policies apply to all profiles and override local and synced settings; they do not delete entries saved
   before (passwords, cards) - remove those once per profile by hand.

7. web apps (`software.webapps`): menu entries that open a site as an app window of a Chromium based browser,
   each in its own browser profile (see below)

Repositories (e.g. Microsoft Edge, OBS projects) are configured by ansible-role-repo, which runs before this role.

## Web apps

A site as its own "app": menu entry with icon, own window without tabs/address bar, and by default its own browser
data dir per user and app - so its cookies and logins are separate from the normal browser (e.g. Telegram's links
don't open where the banking session is), it runs as its own browser process and doesn't show up in the browser's
profile list.

```yaml
software:
  webapps:
    - {id: telegram, name: Telegram, url: "https://web.telegram.org/a/", icon_url: "https://web.telegram.org/a/icon-192x192.png"}
    - {id: element, name: Element, url: "https://app.element.io/", icon_url: "https://app.element.io/vector-icons/180.png"}
    - {id: discord, name: Discord, url: "https://discord.com/app", icon_url: "https://cdn.discordapp.com/embed/avatars/0.png"}
    # one Teams per customer, each in an Edge profile (created by the first start, log in once per laptop)
    - {id: teams-customer1, name: "Teams Customer1", url: "https://teams.microsoft.com/", icon_url: "https://teams.microsoft.com/favicon/prod/favicon-192x192.png", browser: edge, profile: customer1}
    - {id: teams-customer2, name: "Teams Customer2", url: "https://teams.microsoft.com/", icon_url: "https://teams.microsoft.com/favicon/prod/favicon-192x192.png", browser: edge, profile: customer2}
    - {id: oldapp, state: absent}    # uninstall (same as deleting the entry)
```

| field | meaning | default |
|---|---|---|
| `id` | a-z 0-9 _ -, file names, data dir and its profile | required |
| `state` | `present` or `absent` (uninstall; then only `id` is needed) | `present` |
| `name` | menu name | required |
| `url` | http/https, without space `"` `` ` `` `$` `\` | required |
| `browser` | key of `software_browsers` with a `command`: chromium, chrome, brave, edge; must be in the package list | `chromium` |
| `profile` | use this profile of the browser's normal data dir (`--profile-directory`, the directory name from `chrome://version` → Profile Path; a new name creates the profile on the first start) instead of an own data dir | - (own data dir) |
| `icon_url` | png/svg, downloaded once to `webapps_icon_dir/<id>.<ext>` (no .ico: GNOME shows it as noise) | - |
| `icon` | theme icon name if there is no `icon_url` or its download failed | `web-browser` |
| `wm_class` | `StartupWMClass` if the computed one is wrong (see below) | computed |

- Launcher: `webapps_dir/webapp-<id>.desktop` (default `/usr/local/share/applications`, in `XDG_DATA_DIRS`), for all
  users. Home directories are not touched by the role.
  - own data dir: `Exec=<webapps_launcher> <command> <id> "<url>"`; the wrapper (default
    `/usr/local/bin/webapp-launch`, a .desktop file can't expand `$HOME`) runs
    `<command> --user-data-dir=${XDG_DATA_HOME:-~/.local/share}/webapps/<id> --profile-directory=<id> --app=<url>`.
    The browser creates the dir on the first start.
  - with `profile`: `Exec=<command> "--profile-directory=<profile>" "--app=<url>"`.
- Icons: `webapps_icon_dir` (default `/usr/local/share/webapps`); a missing file is downloaded, an existing one is
  kept (delete it to fetch it again). A failed download is a warning, the launcher gets `icon`.
- Launchers carry `X-Webapp-Id/-Browser/-Profile/-Url` keys, read back on removal.
- Uninstall: `state: absent` or entry deleted → launcher and icon removed (other `.desktop` files are not touched),
  the wrapper when the list is empty. The user data in the homes stays (the role never touches homes); on that run
  a cleanup note per web app is shown as "changed" (visible at the default verbosity, also with
  `display_ok_hosts = false`, and so also in ansible.log / the per-host log) and saved on the controller next to
  the ansible log: `<directory of log_path>/cleanup/<inventory_hostname>/web-app-<id>-<YYYYmmdd-HHMMSS>.txt`
  (nsbldr: `/srv/ansible/logs/cleanup/…`; no `log_path` → message only; check mode: message only). Contents:
  - own data dir: per user `rm -rf "${XDG_DATA_HOME:-$HOME/.local/share}/webapps/<id>"` (only this web app's
    cookies, logins, cache) + remove the KeePassXC-Browser key
  - browser profile: no rm - it's a normal browser profile that may hold more; delete it only when unused
    (Edge: `edge://settings/profiles`, Chromium/Chrome/Brave: profile picker)
  - launcher of an older role version (no `X-Webapp-` keys): both possible places
- `StartupWMClass` (GNOME groups the window under the launcher's icon): Chromium names `--app` windows
  `<app_id>-<host>_<path with / as _>-<profile>`, own data dir: profile = id, e.g.
  `chrome-web.telegram.org__a_-telegram`, `msedge-teams.microsoft.com__-customer1`; `app_id` per browser in `vars/main.yml` (chrome, brave, msedge). Check a
  running window in GNOME (Alt+F2 `lg` → Windows, `wmclass`) and set `wm_class` if it differs.
- Policies and extensions are machine-wide, so they apply in the web apps too. KeePassXC-Browser: KeePassXC writes
  its native messaging manifest only into the browser's normal data dir (`~/.config/chromium/NativeMessagingHosts`),
  which a web app with an own data dir doesn't read → when `keepassxc` is in the package list the role writes it
  system-wide for the browsers of these web apps (`nmh_dir` in `vars/main.yml`: Chromium
  `/etc/chromium/native-messaging-hosts`, Chrome `/etc/opt/chrome/…`, Edge `/etc/opt/edge/…`; Brave: none) and
  removes it otherwise. "Connect" in the add-on once per web app.
- Firefox has no app windows on Linux, so it is not offered.

## Packages from one repository (`from`)

For own builds that replace distribution packages (e.g. freerdp from an OBS home project):

```yaml
software:
  packages:
    - {name: freerdp, from: home_hoo}          # zypper alias from ansible-role-repo
    - {name: freerdp-sdl, from: home_hoo}
    - {name: libfreerdp3-3, from: home_hoo}    # list ALL subpackages in use (rpm -qa --qf '%{NAME} %{SOURCERPM}\n')
    - {name: libwinpr3-3, from: home_hoo}
    - {name: librdtk0-0, from: home_hoo}
```

- Check (cached metadata, no network): `zypper --no-refresh search --details --match-exact --repo <alias> <names>`;
  every name with status "installed" (exactly that repository's build) → nothing to do, no zypper transaction -
  also when the repository is unreachable.
- Otherwise all names of that repository in one transaction:
  `zypper --non-interactive install --force --allow-vendor-change --download in-advance --from <alias> <names>`
  (also over the same or a newer version of another vendor; everything is downloaded before the first install). On a conflict zypper aborts and changes nothing; the role shows a warning
  (failed + "...ignoring", counted as `ignored`), the run continues, the next run tries again.
- Afterwards zypper keeps them: different vendor (`obs://build.opensuse.org/home:<user>`), and zypp's defaults
  `solver.allowVendorChange=false` / `solver.dupAllowVendorChange=false` → `zypper up/dup/in` only take updates of
  the same vendor = from that repository; newer distribution versions are ignored. **No repo priority needed** (it
  would prefer every package of the repository, also ones added later).
- Security updates come only from your build: keep the OBS package a branch/link of the distribution package so
  OBS merges its changes and rebuilds.
- A newer build in the repository is installed by `zypper up` (same vendor) or by the next run of the role.
- Back to the distribution packages: remove the entries, then
  `zypper in --force --allow-vendor-change --from <distribution repo alias> <names>`.
- Check: `rpm -q --qf '%{NAME} %{VERSION}-%{RELEASE} %{VENDOR}\n' <names>`

## Variables

```yaml
software:
  languages: []                      # keys of software_langmap: german, french (not on openSUSE yet)
  packages:
    - mc
    - keepassxc
    - microsoft-edge-stable          # needs the repo below
  packages_by_os:                    # names that differ per OS family (ansible_os_family)
    Suse: [diff-pdf]
    Debian: [diffpdf]
    RedHat: [diffpdf]
  chromium_common_policies:          # Chromium, Chrome, Brave, Edge (Chromium policy names)
    PasswordManagerEnabled: false    # e.g. KeePassXC-Browser instead
    AutofillCreditCardEnabled: false
    AutofillAddressEnabled: false
    PaymentMethodQueryEnabled: false
    TranslateEnabled: false
  chrome_store_extensions:           # Chromium, Chrome, Brave
    - ddkjiahejlhfcafbddmgiahcphecmpfh   # uBlock Origin Lite
    - oboonakemofpalcgghocfoadofidjkkk   # KeePassXC-Browser
  edge_store_extensions:             # Edge
    - cimighlppcgcoapaliogpjjdehbnofhn   # uBlock Origin Lite
    - pdffhmdngciaglkoonimfcmckehcpafo   # KeePassXC-Browser
  pin_extensions: true               # toolbar: Chromium/Chrome/Brave toolbar_pin=force_pinned, Edge
                                     # toolbar_state=force_shown, Firefox entries default_area=navbar
  # chromium_policies / chrome_policies / brave_policies / edge_policies: per browser, e.g. BraveRewardsDisabled
  webapps: []                        # see "Web apps"
  webapps_dir: /usr/local/share/applications
  webapps_icon_dir: /usr/local/share/webapps
  webapps_launcher: /usr/local/bin/webapp-launch
  firefox_policies:                  # any Firefox enterprise policy (mozilla.github.io/policy-templates)
    ExtensionSettings:
      "uBlock0@raymondhill.net":
        installation_mode: normal_installed     # force_installed: cannot be disabled either
        install_url: https://addons.mozilla.org/firefox/downloads/latest/ublock-origin/latest.xpi
      "keepassxc-browser@keepassxc.org":
        installation_mode: normal_installed
        install_url: https://addons.mozilla.org/firefox/downloads/latest/keepassxc-browser/latest.xpi
    PasswordManagerEnabled: false
    OfferToSaveLogins: false
    AutofillCreditCardEnabled: false
    AutofillAddressEnabled: false
    TranslateEnabled: false
```

KeePassXC-Browser also needs the browser integration in KeePassXC: Settings → Browser Integration →
enable the browser (once per user; writes the native messaging host file into the user's home). Connecting the
add-on to a database ("Connect" in the add-on, name the connection in KeePassXC) stays manual on purpose: it creates
a key pair per browser profile and database and needs the unlocked database.

Vendor repos (ansible-role-repo), same aliases as `opi msedge` / `opi chrome` / `opi brave`:

```yaml
repo:
  repositories:
    - name: microsoft-edge
      description: "Microsoft Edge (packages.microsoft.com)"
      baseurl: "https://packages.microsoft.com/yumrepos/edge"
      gpgkey: "https://packages.microsoft.com/keys/microsoft.asc"
    - name: google-chrome
      description: "Google Chrome (dl.google.com)"
      baseurl: "https://dl.google.com/linux/chrome/rpm/stable/x86_64"
      gpgkey: "https://dl.google.com/linux/linux_signing_key.pub"
    - name: brave-browser
      description: "Brave Browser (brave-browser-rpm-release.s3.brave.com)"
      baseurl: "https://brave-browser-rpm-release.s3.brave.com/x86_64"
      gpgkey: "https://brave-browser-rpm-release.s3.brave.com/brave-core.asc"
```

Vendor packages with a defaults file: `software_vendor_defaults_files` (defaults/main.yml).

## License

Apache-2.0

Created with the help of AI
