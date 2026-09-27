# ansible-role-software

Installs language packs for the configured languages and a few base packages
(work in progress: the package list is currently fixed in `tasks/main.yml`).

## Variables

```yaml
software:
  languages:          # keys of software_langmap (defaults/main.yml): german, french
    - german
```

Language package name patterns per OS family / distribution are in `vars/<os_family>.yml` and
`vars/<distribution>.yml` (e.g. `langpacks-%s`, `glibc-langpack-%s`).

## License

Apache-2.0

Created with the help of AI
