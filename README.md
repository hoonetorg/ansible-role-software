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
5. `software.packages` plus `software.packages_by_os[<os_family>]` are installed
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

Repositories (e.g. Microsoft Edge, OBS projects) are configured by ansible-role-repo, which runs before this role.

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
