# Fedora Silverblue - Post Installation

A Fedora Silverblue (36) post installation guide.

## Motivation

This repository was created to keep track of the installation and configuration steps (personally) required after a clean installation of Fedora Silverblue.

## Index
To be genenerated...

## Updating the system

Since we've just done a clean install Fedora Silverblue, it would be a good to start off with the latest updates.
```bash
rpm-ostree upgrade --check # dry run to see what will be updated
rpm-ostree upgrade # perform upgrade
systemctl reboot # to use new deployment image
```

> **Note:** Fedora Silverblue inherently needs a reboot after installing updates or new packages via `rpm-ostree`. This is due to the way `os-tree` handles deployment images.

## Updating firmware

At this point, it would be good to retrieve and install the latest firmware for your hardware.
```bash
sudo fwupdmgr get-devices
sudo fwupdmgr refresh --force
sudo fwupdmgr get-updates
sudo fwupdmgr update
systemctl reboot
```

## Configuring DNF

Since DNF is not particularly fast out of the box, we can make some adjustments in its configuration.

```bash
echo 'fastestmirror=1' | sudo tee -a /etc/dnf/dnf.conf
echo 'max_parallel_downloads=10' | sudo tee -a /etc/dnf/dnf.conf
echo 'deltarpm=true' | sudo tee -a /etc/dnf/dnf.conf
cat /etc/dnf/dnf.conf # see current configuration
```

## Setting the hostname

```bash
hostnamectl set-hostname fedora-sb
```

## Package layering

Fedora Silverblue uses `os-tree` to layer packages over the base system. This way, rollbacks and upgrades can be performed on the base system while keeping the layered packages as they were. Moreover, rollbacks can be performed to deployments where a (possibly problematic) layered package had not been layered yet.

### Enabling RPM Fusion repositories
It was assumed that the RPM Fusion repositories have been enabled prior to the following steps.

If not enabled yet, run:
```bash
rpm-ostree install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm # free packages
rpm-ostree install https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm # non-free packages
systemctl reboot #to use new deployment image
```

### NVIDIA driver installation (with CUDA)

```bash
# Install NVIDIA driver
rpm-ostree install akmod-nvidia xorg-x11-drv-nvidia # NVIDIA
rpm-ostree install xorg-x11-drv-nvidia-CUDA # CUDA
rpm-ostree kargs --append=rd.driver.blacklist=nouveau --append=modprobe.blacklist=nouveau --append=nvidia-drm.modeset=1 # Blacklist nouveau driver
systemctl reboot #to use new deployment image
```

### Other packages

```bash
rpm-ostree install ulauncher neofetch lm_sensors dkms mscore-fonts-all gnome-tweaks
systemctl reboot #to use new deployment image
```

> **Note:** To enable ULauncher short-cut to be `Super + /`, add a custom keyboard shortcut for command `ulauncher-toggle` in GNOME settings.

> **Note:** If the microsoft fonts installed with `mscore-fonts-all` are not being picked up properly. Try copying the fonts from `/usr/share/font` to `~/.local/share/fonts`.

## Fish shell

To install the Fish shell and make it the default shell, run:

```bash
rpm-ostree install fish
sudo lchsh $USER # to /bin/fish
systemctl reboot #to use new deployment image
```

To install custom plugins (using Fisher), run:

```bash
curl -sL https://git.io/fisher | source && fisher install jorgebucaran/fisher
fisher install IlanCosman/tide@v5
fisher install franciscolourenco/done
fisher install jorgebucaran/autopair.fish
```

To change the default colours in GNOME Terminal (using Gogh), run:

```bash
bash -c "$(wget -qO- https://git.io/vQgMr)"
```

## Configuring Flatpak

In Fedora Silverblue, Flatpak is enabled out of the box. However, the Flathub repository may or may not yet be added yet:

```bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

Next, update the Flatpak apps and install Flatseal (to manage app permissions):

```bash
flatpak update
flatpak install flathub com.github.tchx84.Flatseal
```

To enable the Flatpak apps to use Adwaita-dark (or any other GTK theme), run:

```bash
mkdir ~/.themes
cp -R /usr/share/themes/* ~/.themes
sudo flatpak override --filesystem=~/.themes
sudo flatpak override --env=GTK_THEME=Adwaita-dark
```

## Firefox

To install Firefox Nightly and remove the pre-installed Firefox version, run:

```bash
flatpak install https://gitlab.com/proletarius101/firefox-nightly-flatpak/raw/master/firefox-nightly.flatpakref
rpm-ostree override remove firefox
systemctl reboot
```

> **Note:** While this may work out of the box, some issues with X11 can occur (especially when spawning new windows). This may be fixed by adding an environment variable. For Fish add `set -gx MOZ_DBUS_REMOTE 1` to the configuration file (e.g. `~/.config/fish/config.fish`).

## Toolbox

Toolboxes will be used as development environments. They'll contain the `dnf` packages which are field/project/toolbox specific. Toolboxes can simply be destroyed and recreated when problems arise.

### Constructing a base image

Since there may be packages which all toolboxes will need/use, it would be nice to construct a base image to create future toolboxes from. First, we create a base toolbox:

```bash
toolbox create fedora36-base
toolbox enter fedora36-base
```

Next, we'll install some packages which may be useful for all future toolboxes (we can use `dnf` now):

```bash
sudo dnf groupinstall "Development Tools"
sudo dnf groupinstall "C Development Tools and Libraries"
sudo dnf install fish # already installed with rpm-ostree, but this avoids toolbox issues
```

To create an image out of this toolbox, first exit the container using `Ctrl + D`, then run:

```bash
podman stop fedora36-base
podman commit fedora36-base
podman images # note the added image
podman tag <IMAGE_ID> fedora36-base # for convenience
toolbox rm fedora36-base # can be removed now
```

### Using the base image

To create a toolbox by extending from the base image, the `--image` flag can be used.

As an example, we can create a toolbox for working with LaTeX:

```bash
toolbox create --image fedora36-base latex
toolbox enter latex
sudo dnf install -y texlive-scheme-full latexmk
```

As another example, we can create a toolbox for working with AI stuff:

```bash
toolbox create --image fedora36-base AI
toolbox enter AI
sudo dnf install python3 python3-pip python3-matplotlib python3-notebook python3-numpy \
    python3-pandas python3-pillow python3-scikit-image python3-scikit-learn python3-scipy \
    python3-ipykernel python3-ipython python3-jupyter-client python3-jupyter-core
pip3 install torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu116
```

## Installing Flatpak Apps

The following script installs quite some applications may be useful.

```bash
flatpak install flathub org.gnome.Extensions # GNOME customization
flatpak install flathub com.mattjakeman.ExtensionManager # GNOME customization
flatpak install flathub com.bitwarden.desktop # Password manager
flatpak install flathub nz.mega.MEGAsync # File sync between devices
flatpak install flathub md.obsidian.Obsidian # Taking notes with enhanced markdown
flatpak install flathub org.onlyoffice.desktopeditors # Office suite of choice
flatpak install flathub org.videolan.VLC # Codecs
flatpak install flathub com.discordapp.Discord
flatpak install flathub org.gimp.GIMP # Image editing
flatpak install flathub org.inkscape.Inkscape # SVG/Image editing
flatpak install flathub com.transmissionbt.Transmission # Torrents
flatpak install flathub org.gnome.Evolution # Email
flatpak install flathub io.github.mimbrero.WhatsAppDesktop
flatpak install flathub com.spotify.Client # Music
flatpak install flathub com.visualstudio.code # Coding
```

## Installing Gnome Extensions

A list of extensions which could be useful:
- AppIndicator and KStatusNotifierItem Support
- Extension list
- franciscolourenco
- NoAnnoyance v2
- OpenWeather
- Removable Drive Menu
- Sound Input & Output Device Chooser
- Todo.txt

Next to that, it would be convenient to set the window/app-switcher to detect only the apps in the current workspace:

```bash
gsettings set org.gnome.shell.window-switcher current-workspace-only true
gsettings set org.gnome.shell.app-switcher current-workspace-only true
```

## Adding aliases

Generally, it would be useful to have aliases for toolboxes and Visual Studio Code. An example setup would add the following lines to the Fish configuration file (e.g. `~/.config/fish/config.fish`):

```bash
# Previously constructed toolboxes
alias tb-ai="toolbox enter AI"
alias tb-latex="toolbox enter latex"

# VSCode flatpak
alias code="flatpak run com.visualstudio.code"
```

## Adding GPG- and SSH-key to GitHub


### GPG-key

Add a GPG-key (for signing commits) using:

```bash
gpg --full-generate-key
gpg --list-secret-keys --keyid-format=long
gpg --armor --export <KEY_ID>
git config --global user.signingkey <KEY_ID>
```

Then, add the following line to the Fish configuration file (e.g. `~/.config/fish/config.fish`):

```bash
set -gx GPG_TTY (tty)
```

Lastly, enter the output of the following command in GitHub:

```bash
gpg --armor --export <KEY_ID>
```

### SSH-key

Add a SSH-key using:

```bash
ssh-keygen -t ed25519 -C "<account_email>"
eval (ssh-agent -c)
ssh-add ~/.ssh/id_ed25519
```

Lastly, enter the output of the following command in GitHub:

```bash
cat ~/.ssh/id_ed25519.pub
```

## Enabling RTL88x2BU WiFi drivers

In Fedora Silverblue, the WiFi driver installation is more of a pain than for Fedora Workstation. A solution was found where the `ostree` image is hotfixed with the WiFi drivers. However, this script needs to be re-run after every new image deploy, since the hotfix get's "reset". Fortunately, after the initial setup of the system, this shouldn't have to be done often.

```bash
sudo ostree admin unlock --hotfix # ugly but working
sudo git clone "https://github.com/RinCat/RTL88x2BU-Linux-Driver.git" /usr/src/rtl88x2bu-git
sudo sed -i 's/PACKAGE_VERSION="@PKGVER@"/PACKAGE_VERSION="git"/g' /usr/src/rtl88x2bu-git/dkms.conf
sudo dkms remove -m rtl88x2bu -v git # if already present
sudo dkms add -m rtl88x2bu -v git # add driver
sudo dkms autoinstall # install driver
systemctl reboot # to load in driver
```

These commands can be put into a script (e.g. `install_wifi_drivers.sh`) and run when the deployment image has changed.

## More resources
- https://silverblue.fedoraproject.org/
- https://docs.fedoraproject.org/en-US/fedora-silverblue/installation/
- https://docs.fedoraproject.org/en-US/fedora-silverblue/_attachments/flatpak-print-cheatsheet.pdf
- https://docs.fedoraproject.org/en-US/fedora-silverblue/_attachments/silverblue-cheatsheet.pdf
- https://rpmfusion.org/Howto/NVIDIA?highlight=%28%5CbCategoryHowto%5Cb%29#Silverblue
- https://mutschler.dev/linux/fedora-post-install/
- https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent
- https://docs.github.com/en/authentication/managing-commit-signature-verification/generating-a-new-gpg-key
