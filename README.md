# ansible-role-syncthing

Runs [Syncthing](https://syncthing.net/) 2.x as `syncthing@<user>.service` with a fixed identity
(cert/key from the inventory → stable device ID), GUI user/password, API key and HTTPS, and manages
devices and folders through the REST API.

Order of operations:
1. install the package, assert version >= `min_version` (on openSUSE Leap add the OBS `network` repository,
   e.g. with ansible-role-repo: `https://download.opensuse.org/repositories/network/<version>/`)
2. `cert.pem` / `key.pem` into the syncthing home (`<user home>/.local/state/syncthing`)
3. `syncthing generate --gui-user … --gui-password=-` creates `config.xml` (only if missing)
4. systemd drop-in (`STHOMEDIR`), enable + start `syncthing@<user>.service`
5. REST API: GUI (address, TLS, user, API key, password), devices, folders
6. firewalld service `syncthing` (if firewalld is running)

The user must exist (run after ansible-role-user).

## Requirements

- controller: collections `ansible.utils`, `community.general`, `ansible.posix`; Python `xmltodict`
  (`ansible.utils.from_xml` parses syncthing's `config.xml`)
- target: syncthing >= 2.0, systemd, `runuser` (util-linux)

## Variables

```yaml
syncthing:
  user: alice                                   # required
  name: alice-laptop                            # own device name, default: inventory_hostname_short
  # home: /home/alice/.local/state/syncthing    # default
  cert: "{{ vault_syncthing_cert }}"            # required, PEM
  key: "{{ vault_syncthing_key }}"              # required, PEM
  apikey: "{{ vault_syncthing_apikey }}"        # required
  gui:
    address: "127.0.0.1:8384"                   # default, localhost only
    tls: true                                   # default
    user: alice                                 # default: syncthing.user
    password: "{{ vault_syncthing_gui_password }}"   # required, plain text (stored as bcrypt hash)
  firewalld: true                               # default
  devices:
    - id: "AAAAAAA-BBBBBBB-CCCCCCC-DDDDDDD-EEEEEEE-FFFFFFF-GGGGGGG-HHHHHHH"
      name: phone
      introducer: false                         # default false
      auto_accept_folders: false                # default false
      # state: absent                           # remove the device
  folders:
    - id: docs-abcd1                            # must match the folder ID on the other devices
      label: Documents                          # default: id
      path: ~/Documents                         # or an absolute path
      type: sendreceive                         # sendreceive | sendonly | receiveonly | receiveencrypted
      devices: [phone]                          # device names from syncthing.devices
      devices_exclusive: false                  # default: syncthing.folder_devices_exclusive (false)
      # state: absent                           # remove the folder (files stay on disk)
```

Folder paths are passed unchanged to syncthing: `~` is expanded by syncthing itself (home of
`syncthing.user`), and syncthing creates missing directories (incl. parents) as that user. Paths outside the
user's writable area (e.g. `/srv/...`) must exist beforehand (e.g. created by ansible-role-disk).

Devices and folders not listed are left untouched. New objects get syncthing's defaults for all other
settings.

Folder sharing: with `devices_exclusive: false` the listed devices are only added (devices shared manually or
by an introducer stay); with `devices_exclusive: true` the folder is shared with exactly the listed devices plus
the own device, all others are removed from the folder (the devices themselves stay configured). On an
introducer such removals propagate to the devices that use it as introducer. The role-wide default is
`syncthing.folder_devices_exclusive` (false).

The own device name (`syncthing.name`) is checked on every run and set via the API if it differs
(`syncthing generate` uses the hostname of that moment, e.g. a DHCP FQDN).

## Identity

The device ID is derived from `cert.pem`. Create a new identity and put both files into the vault:

```bash
syncthing generate --home=/tmp/st-new      # prints the device ID
cat /tmp/st-new/cert.pem /tmp/st-new/key.pem
rm -rf /tmp/st-new
```

## Introducer

The introducer flag is set on the *other* devices for the device that should introduce (edit the device
there and enable "Introducer"); nothing has to be configured on the introducer itself.

## License

Apache-2.0

Created with the help of AI
