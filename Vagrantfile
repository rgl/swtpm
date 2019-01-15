Vagrant.configure('2') do |config|
    config.vm.box = 'ubuntu-18.04-amd64'

    config.vm.provider :libvirt do |lv, config|
      lv.memory = 1024
      lv.cpus = 2
      lv.cpu_mode = 'host-passthrough'
      lv.nested = true
      lv.keymap = 'pt'
      config.vm.synced_folder '.', '/vagrant', type: 'nfs'
    end

    config.vm.provision :shell, inline: <<-PROVISION_SCRIPT
set -eux
export DEBIAN_FRONTEND=noninteractive
apt-get update

# install qemu-kvm.
apt-get install -y p7zip-full
apt-get install -y qemu-kvm
apt-get install -y sysfsutils
systool -m kvm_intel -v

# install swtpm.
apt-get install -y dpkg-dev
(cd /vagrant/build && dpkg-scanpackages . | gzip -9c >Packages.gz)
echo ''deb [trusted=yes] file:/vagrant/build ./'' >/etc/apt/sources.list.d/swtpm.list
apt-get update
apt-get install -y swtpm swtpm-tools
swtpm --version

# initialize a tpm.
# see man swtpm_setup
export XDG_CONFIG_HOME=~/.config
mkdir -p $XDG_CONFIG_HOME
cat >$XDG_CONFIG_HOME/swtpm_setup.conf <<'EOF'
# Program invoked for creating certificates
create_certs_tool= /usr/share/swtpm/swtpm-localca
create_certs_tool_config = ${XDG_CONFIG_HOME}/swtpm-localca.conf
create_certs_tool_options = ${XDG_CONFIG_HOME}/swtpm-localca.options
EOF
cat >$XDG_CONFIG_HOME/swtpm-localca.conf <<'EOF'
statedir = ${XDG_CONFIG_HOME}/var/lib/swtpm-localca
signingkey = ${XDG_CONFIG_HOME}/var/lib/swtpm-localca/signkey.pem
issuercert = ${XDG_CONFIG_HOME}/var/lib/swtpm-localca/issuercert.pem
certserial = ${XDG_CONFIG_HOME}/var/lib/swtpm-localca/certserial
EOF
cat >$XDG_CONFIG_HOME/swtpm-localca.options <<'EOF'
--platform-manufacturer Ubuntu
--platform-model QEMU
--platform-version 2.11
EOF
mkdir -p ${XDG_CONFIG_HOME}/mytpm1
swtpm_setup \
  --tpm2 \
  --tpmstate ${XDG_CONFIG_HOME}/mytpm1 \
  --create-ek-cert \
  --create-platform-cert \
  --lock-nvram
# TODO run swtpm as a systemd service (without --daemon) and as a non-previleged user and log to stdout.
swtpm \
  socket \
  --daemon \
  --tpmstate dir=${XDG_CONFIG_HOME}/mytpm1 \
  --ctrl type=unixio,path=${XDG_CONFIG_HOME}/mytpm1/swtpm-sock \
  --log file=${XDG_CONFIG_HOME}/mytpm1.log,level=20

# download the alpine iso to the shared storage.
alpine_iso=alpine-virt-3.8.2-x86_64.iso # NB this does not have the tpm modules.
alpine_iso=alpine-standard-3.8.2-x86_64.iso
alpine_iso_path=/vagrant/tmp/$alpine_iso
if [[ ! -f $alpine_iso_path ]]; then
    mkdir -p $(dirname $alpine_iso_path)
    wget -qO $alpine_iso_path http://dl-cdn.alpinelinux.org/alpine/v3.8/releases/x86_64/$alpine_iso
fi
if [[ ! -f $alpine_iso ]]; then
  cp $alpine_iso_path $alpine_iso
  7z x $alpine_iso boot/vmlinuz-vanilla boot/initramfs-vanilla
fi
cat <<'EOF'
run a vm with:

  qemu-system-x86_64 \
    -enable-kvm \
    -nographic \
    -cdrom $alpine_iso \
    -kernel boot/vmlinuz-vanilla \
    -initrd boot/initramfs-vanilla \
    -append 'modules=loop,squashfs,sd-mod,tpm nomodeset console=ttyS0' \
    -net nic \
    -net user \
    -m 256 \
    -rtc base=utc \
    -chardev socket,id=chrtpm,path=${XDG_CONFIG_HOME}/mytpm1/swtpm-sock \
    -tpmdev emulator,id=tpm0,chardev=chrtpm \
    -device tpm-tis,tpmdev=tpm0

type "ctrl-a h" to see the qemu emulator help
type "ctrl-a x" to quit the qemu emulator
see https://github.com/qemu/qemu/blob/master/docs/specs/tpm.txt
EOF
PROVISION_SCRIPT
end
