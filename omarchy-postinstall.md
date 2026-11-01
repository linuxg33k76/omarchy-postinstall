# Omarchy post-install

Stand-alone playbook for a fresh Omarchy workstation. Do the phases in order. Do not invent extra packages, plugins, themes, or desktop tweaks. This file is the whole procedure. Do not look for another script, YAML file, or repository to interpret it.

## Rules

- Run as the desktop user, never as root. Use `sudo` for privileged commands. Do not use `pkexec`.
- Do not edit anything under `/usr/share/omarchy/`. Read it if you need to understand a command.
- Do not install packages, plugins, or themes that are not named in this file.
- Do not copy wallpapers or clone a wallpaper repository. The desktop background comes from the built-in Osaka Jade theme.
- Do not install files from `~/Downloads`.
- Do not edit PAM, SDDM, or fingerprint configuration by hand. Fingerprint setup is the Omarchy wizard in Phase 0, and only when the user asks for it.
- Do not reboot, shut down, or change the hostname, git identity, or Samba mount until the user has answered the questions in Phase 0.
- Confirm each phase succeeded before starting the next one. If a command fails, stop that phase, report the error, and do not continue past it.
- Append a one-line result per phase to `~/omarchy-postinstall.log`. Never write a Samba password, git credential, or SSH private key into the log or into this file.
- `omarchy plugin add` prompts unless you pass `--yes`. Always pass `--enable --yes`. These plugins run unsandboxed inside the Omarchy shell. The user already chose this list.

## Phase 0 — Ask once, then proceed

Collect all of the following before changing the system.

1. **Go ahead.** User wants this full post-install on this Omarchy machine.
2. **Skip set.** Any of `update`, `install`, or `all`. Default: skip nothing. `all` skips the two update phases and the package install. Backup, plugins, theme, and the later phases still run.
3. **Backup directory.** Default: `~/backup`.
4. **Samba.** Mount a remote CIFS share at `/mnt/remote_cifs`, or skip. If yes, collect and confirm: server IP, server hostname, share name, username, password (typed twice), domain. Show the values back with the password masked and wait for an explicit yes.
5. **Git.** Set a global `user.name` and `user.email`, or skip. If yes, show the current values, collect new ones, and confirm.
6. **Hostname.** New hostname, or keep the current one. Empty means keep.
7. **SSH.** If no public key exists, ask the user for the backup ssh-key locatin, generate an Ed25519 key or skip. If generating, the user types the passphrase in the terminal. Do not invent an empty passphrase.
8. **Fingerprint.** Default: skip. Only if the user explicitly wants it, run `omarchy setup security fingerprint` and let them finish that wizard. Ask user if they wish to set it up.

Refuse to start unless step 1 is yes. Record the other answers for later phases.

## Phase 1 — Preflight

```bash
whoami
. /etc/os-release && printf '%s %s id=%s like=%s\n' "$PRETTY_NAME" "$VERSION_ID" "$ID" "$ID_LIKE"
uname -a
command -v yay
command -v omarchy
ping -c 1 -W 3 archlinux.org
```

Stop if any of these are true:

- `whoami` prints `root`
- `ID` is not `omarchy` and `ID_LIKE` does not contain `arch`
- `yay` or `omarchy` is missing
- The ping fails

Create the working directories. `${BACKUP}` is the Phase 0 backup directory. `~/code` is only for the `bash_scripts` checkout in Phase 6.

```bash
mkdir -p "${BACKUP}" "${HOME}/code"
test -d "${BACKUP}" && test -d "${HOME}/code"
```

Log the OS line and the skip set.

## Phase 2 — Update

Skip this phase if Phase 0 included `update` or `all`.

Use `--noconfirm` so the update does not wait on a prompt. If pacman stops on a package replacement or conflict, stop and ask the user. Do not answer the conflict yourself.

```bash
sudo pacman -Syyu --noconfirm
```

## Phase 3 — Packages

Skip this phase if Phase 0 included `install` or `all`.

`--needed` skips packages that are already installed.

Official repositories:

```bash
sudo pacman -S --needed --noconfirm \
  sysfsutils \
  stow \
  solaar \
  deja-dup \
  rsync \
  rclone \
  yazi
```

AUR:

```bash
yay -S --needed --noconfirm \
  google-chrome \
  visual-studio-code-bin \
  insync
```

If `yay` cannot reach the AUR, stop. Do not switch package names. `omarchy pkg aur add` is an acceptable substitute only for those same six names, and only after `omarchy pkg aur accessible` succeeds.

Verify every name is installed:

```bash
pacman -Q sysfsutils stow solaar deja-dup rsync rclone yazi \
  google-chrome \
  visual-studio-code-bin insync
```

`pacman -Q` is the right check for AUR packages too. A missing name is a failed install, not a warning to ignore.

## Phase 4 — Update again

Skip this phase if Phase 0 included `update` or `all`.

Refresh once more so packages installed in Phase 3 are current.

```bash
sudo pacman -Syyu --noconfirm
```

## Phase 5 — Back up files later phases may change

```bash
stamp=$(date '+%m-%d-%Y-%H_%M_%S')
sudo cp -a /etc/fstab "/etc/fstab.bak.${stamp}"
sudo cp -a /etc/pam.d/sudo "/etc/pam.d/sudo.bak.${stamp}"
```

Confirm both `.bak.${stamp}` files exist before Phase 7. Phase 7 is the only phase that writes `/etc/fstab`. Do not edit `/etc/pam.d/sudo` in this playbook. The copy is a rollback point if the fingerprint wizard is used.

## Phase 6 — Environment

### bash_scripts

Clone only when `~/code/bash_scripts` is missing.

```bash
if [ ! -d "${HOME}/code/bash_scripts/.git" ]; then
  git clone https://github.com/linuxg33k76/bash_scripts.git "${HOME}/code/bash_scripts"
fi
```

Leave an existing checkout alone. Do not pull, reset, or change its branch.

### Shell plugins

Add each repo with `--enable --yes`. If a plugin is already installed, skip that URL and say so. Do not remove or update plugins that are already present.

```bash
omarchy plugin add https://github.com/HoneySpoons/omarchy-localsend.git --enable --yes
omarchy plugin add https://github.com/r-witz/omarchy-nothing-ear.git --enable --yes
omarchy plugin add https://github.com/ronald2wing/Omarchy-Bible --enable --yes
omarchy plugin add https://github.com/linuxg33k76/omanetmonitor.git --enable --yes
omarchy plugin add https://github.com/linuxg33k76/hey-im-gaming-here.git --enable --yes
omarchy plugin add https://github.com/fross100/omaplug.git --enable --yes
```

Then check that they are present and enabled:

```bash
omarchy plugin list
```

Match on the plugin id from each repo's `manifest.json` if the list does not show the git URL. A failed clone stops the plugin portion. Leave the ones that succeeded installed, report the URL that failed, and continue to Phase 7 only if the user says to.

## Phase 7 — Samba

Skip unless Phase 0 was an explicit yes.

Use the confirmed IP, hostname, share, username, domain, and password. `${BACKUP}` is the Phase 0 directory. Replace the angle-bracket placeholders. Do not put the password in the shell command line or the log.

Write `~/.sambacreds` with mode `0600` before it contains the password.

```bash
install -m 600 /dev/null "${HOME}/.sambacreds"
```

Write these three lines, and nothing else, into that file:

```text
username=<samba user>
password=<samba password>
domain=<samba domain>
```

Confirm the mode is `600`, then copy it into the backup directory:

```bash
stat -c '%a %n' "${HOME}/.sambacreds"
cp -a "${HOME}/.sambacreds" "${BACKUP}/.sambacreds"
chmod 600 "${BACKUP}/.sambacreds"
```

Add the host only if that IP or hostname is not already in `/etc/hosts`:

```bash
grep -E "<ip>|<hostname>" /etc/hosts || echo "<ip>  <hostname>" | sudo tee -a /etc/hosts
```

Create the mount point and give it to the desktop user:

```bash
sudo mkdir -p /mnt/remote_cifs
sudo chmod 770 /mnt/remote_cifs
sudo chown "${USER}:${USER}" /mnt/remote_cifs
id -u
id -g
```

Append the fstab entries only if the share is not already listed. Use `file_mode=0755` and `dir_mode=0755`.

```bash
grep -q '# Remote CIFS mount' /etc/fstab \
  || echo '# Remote CIFS mount' | sudo tee -a /etc/fstab

grep -q "//<hostname>/<share>" /etc/fstab \
  || echo "//<hostname>/<share>  /mnt/remote_cifs  cifs  credentials=${HOME}/.sambacreds,uid=<uid>,gid=<gid>,file_mode=0755,dir_mode=0755,nofail,_netdev  0  0" \
  | sudo tee -a /etc/fstab
```

`<uid>` and `<gid>` are the numbers from `id -u` and `id -g`.

Mount and check:

```bash
sudo systemctl daemon-reload
sudo mount -a
findmnt /mnt/remote_cifs
```

If `mount -a` fails, restore `/etc/fstab` from the Phase 5 backup, tell the user, and continue with Phase 8. Do not keep retrying with altered options.

## Phase 8 — Package inventory

The date is in the filename, so a later run does not overwrite today's list.

```bash
pacman -Qe > "${BACKUP}/Installed_Arch_Packages_$(date +%m_%d_%Y).log"
```

Do not dump dconf or other desktop settings. Omarchy runs Hyprland.

## Phase 9 — Theme

Apply the built-in Osaka Jade theme. Do not install a theme from git, do not copy backgrounds into `~/.config/omarchy/backgrounds/`, and do not edit `/usr/share/omarchy/themes/osaka-jade`.

Confirm the theme is installed, apply it, and check the result:

```bash
omarchy theme list
omarchy theme set osaka-jade
omarchy theme current
```

`omarchy theme list` must include `Osaka Jade` before you run `omarchy theme set`. `omarchy theme current` must print `Osaka Jade` afterward. If the name is missing from the list, stop this phase and report that. Do not pick a different theme.

`osaka-jade` and `Osaka Jade` are the same theme. Use `osaka-jade`.

## Phase 10 — Git identity

Skip unless Phase 0 was an explicit yes. Use the confirmed name and email.

```bash
git config --global user.name "<name>"
git config --global user.email "<email>"
git config --global --get user.name
git config --global --get user.email
```

Do not change `init.defaultBranch`, signing keys, or any other git setting.

## Phase 11 — Hostname

Skip if Phase 0 said to keep the current name.

```bash
hostnamectl
sudo hostnamectl set-hostname "<hostname>"
hostnamectl
```

Accept a single hostname label: letters, digits, and internal hyphens. If the value has spaces or slashes, stop and ask again. Do not edit `/etc/hosts` for the new hostname.

## Phase 12 — SSH key

Look for an existing public key:

```bash
ls -1 "${HOME}/.ssh/"*.pub 2>/dev/null || true
```

If one exists, print the public file path and stop this phase. Do not make a second key.

If none exists and Phase 0 said to generate one, run this in the terminal so the user can enter a passphrase:

```bash
mkdir -p "${HOME}/.ssh"
chmod 700 "${HOME}/.ssh"
ssh-keygen -t ed25519 -f "${HOME}/.ssh/id_ed25519"
```

Do not pass `-N` with a password you chose. After it returns, print the public key only:

```bash
cat "${HOME}/.ssh/id_ed25519.pub"
```

If Phase 0 said to skip, do nothing.

## Phase 13 — Done

Report these and stop. Do not reboot.

- Backup directory path, the two `/etc` backup filenames, and the package-list filename.
- `pacman -Q` result for the Phase 3 packages.
- Which plugins were added, and any URL that failed.
- Whether Samba is mounted (`findmnt /mnt/remote_cifs`), or that it was skipped.
- `omarchy theme current` (expect `Osaka Jade`).
- `git config --global --get` name and email, or that git was skipped.
- Hostname.
- Public key path, or that SSH was skipped.
- A reminder that a kernel upgrade from Phase 2 or Phase 4 needs a reboot before it is running. Let the user reboot.

## Out of scope

Leave these alone.

- Hyprland bindings, bar layout, and other files under `~/.config/hypr/` or `~/.config/omarchy/` beyond what `omarchy plugin add` and `omarchy theme set` write
- Stock theme files under `/usr/share/omarchy/themes/`
- Extra themes, wallpaper repositories, and manual background changes
- A second run of package installs that tries to add software this file does not name
