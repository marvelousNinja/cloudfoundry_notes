cloudfoundry_notes
==================

```
sudo mkdir /mnt/stemcells  
sudo mount --bind ~/tmp/stemcells /mnt/stemcells

rake release:create_dev_release
rake ci:build_stemcell[openstack,ubuntu,ruby]
```

https://help.ubuntu.com/community/Apt-Cacher-Server

What is berkshelf?

git clone --recursive git://github.com/foo/bar.git
OR
git submodule init 
git submodule update

I needed to install these packages:
kpartx
debootstrap
qemu

Pull request with some details:
https://github.com/Altoros/bosh/pull/3

Google's guide to creating custom images:
https://developers.google.com/compute/docs/building-image

The process looks like this:

Here's the rake task for building stemcells:
```ruby
desc 'Build a stemcell for the given :infrastructure, and :operating_system and copy to ./tmp/'
  task :build_stemcell, [:infrastructure_name, :operating_system_name, :agent_name] do |_, args|
    require 'bosh/dev/stemcell_builder'

    stemcell_builder = Bosh::Dev::StemcellBuilder.for_candidate_build(
      args.infrastructure_name, args.operating_system_name, args.agent_name)
    stemcell_file = stemcell_builder.build_stemcell

    mkdir_p('tmp')
    cp(stemcell_file, File.join('tmp', File.basename(stemcell_file)))
  end
end
```

So, after collecting parameters we call StemcellBuilder.for_candidate_build:
```ruby
class StemcellBuilder
  def self.for_candidate_build(infrastructure_name, operating_system_name, agent_name)
    new(
      ENV.to_hash,
      Build.candidate,
      infrastructure_name,
      operating_system_name,
      agent_name
    )
  end

  def initialize(env, build, infrastructure_name, operating_system_name, agent_name)
    @build_number = build.number
    @stemcell_builder_command = Bosh::Stemcell::BuilderCommand.new(
      env,
      infrastructure_name: infrastructure_name,
      operating_system_name: operating_system_name,
      agent_name: agent_name,
      version: build.number,
      release_tarball_path: build.release_tarball_path,
    )
  end
end
```

And finally - StemcellBuilder#build_stemcell:
```ruby
class StemcellBuilder
  def build_stemcell
    gem_components = GemComponents.new(@build_number)
    gem_components.build_release_gems

    @stemcell_path = stemcell_builder_command.build

    File.exist?(@stemcell_path) || raise("#{@stemcell_path} does not exist")

    @stemcell_path
  end
end
```

That leads us to BuilderCommand#build (and constructor):
```ruby
class BuilderCommand
  def initialize(env, options)
    @environment = env
    @infrastructure = Infrastructure.for(options.fetch(:infrastructure_name))
    @operating_system = OperatingSystem.for(options.fetch(:operating_system_name))
    @agent_name = options.fetch(:agent_name) || 'ruby'

    @stemcell_builder_options = BuilderOptions.new(
      env,
      tarball: options.fetch(:release_tarball_path),
      stemcell_version: options.fetch(:version),
      infrastructure: infrastructure,
      operating_system: operating_system,
      agent_name: agent_name,
    )
    @shell = Bosh::Core::Shell.new
  end

  def build
    sanitize

    prepare_build_root

    prepare_build_path

    copy_stemcell_builder_to_build_path

    prepare_work_root

    persist_settings_for_bash

    stage_collection = StageCollection.new(
      infrastructure: infrastructure,
      operating_system: operating_system,
      agent_name: agent_name,
    )
    stage_runner = StageRunner.new(
      build_path: build_path,
      command_env: command_env,
      settings_file: settings_path,
      work_path: work_root
    )
    stage_runner.configure_and_apply(stage_collection.all_stages)
    system(rspec_command) || raise('Stemcell specs failed')

    stemcell_file
  end
end
```

It seems that stage_collection and stage_runner interaction is the only thing, dependable on infrastructure and operating system parameters.
So, we dive in StageCollection and StageRunner classes. Here's all we need to know about StageCollection:
```ruby
class StageCollection
  def initialize(options)
    @infrastructure = options.fetch(:infrastructure)
    @operating_system = options.fetch(:operating_system)
    @agent_name = options.fetch(:agent_name)
  end

  def all_stages
    operating_system_stages + agent_stages + infrastructure_stages
  end

  def infrastructure_stages
    case infrastructure
      when Infrastructure::Aws then
        aws_stages
      when Infrastructure::OpenStack then
        openstack_stages
      when Infrastructure::Vsphere then
        operating_system.instance_of?(OperatingSystem::Centos) ? hacked_centos_vsphere : vsphere_stages
    end
  end

  def aws_stages
    [
      # Misc
      :system_aws_network,
      :system_aws_modules,
      :system_parameters,
      # Finalisation
      :bosh_clean,
      :bosh_harden,
      :bosh_harden_ssh,
      # Image/bootloader
      :image_create,
      :image_install_grub,
      :image_aws_update_grub,
      :image_aws_prepare_stemcell,
      # Final stemcell
      :stemcell
    ]
  end

  def openstack_stages
    [
      # Misc
      :system_openstack_network,
      :system_openstack_clock,
      :system_openstack_modules,
      :system_parameters,
      # Finalisation,
      :bosh_clean,
      :bosh_harden,
      :bosh_harden_ssh,
      # Image/bootloader
      :image_create,
      :image_install_grub,
      :image_openstack_qcow2,
      :image_openstack_prepare_stemcell,
      # Final stemcell
      :stemcell_openstack
    ]
  end
end
```

Here it is. First notable differences. We have a collection of stages for each infrastructure. Same goes for operating system.
Let's switch to StageRunner. It will show us how these steps are interpretered:
```ruby
class StageRunner
  def initialize(options)
    @build_path = options.fetch(:build_path)
    @command_env = options.fetch(:command_env)
    @settings_file = options.fetch(:settings_file)
    @work_path = options.fetch(:work_path)
  end

  def configure_and_apply(stages)
    configure(stages)
    apply(stages)
  end

  def configure(stages)
    stages.each do |stage|
      stage_config_script = File.join(build_path, 'stages', stage.to_s, 'config.sh')

      puts "=== Configuring '#{stage}' stage ==="
      if File.exists?(stage_config_script) && File.executable?(stage_config_script)
        run_sudo_with_command_env("#{stage_config_script} #{settings_file}")
      end
    end
  end

  def apply(stages)
    work_directory = File.join(work_path, 'work')

    stages.each do |stage|
      FileUtils.mkdir_p(work_directory)

      puts "=== Applying '#{stage}' stage ==="
      puts "== Started #{Time.now.strftime('%a %b %e %H:%M:%S %Z %Y')} =="

      stage_apply_script = File.join(build_path, 'stages', stage.to_s, 'apply.sh')

      run_sudo_with_command_env("#{stage_apply_script} #{work_directory}")
    end
  end
end
```

Well, right now I'm trying to understand... All these steps ('stages') for each infrastructure, are they needed for Google Computational Engine too?
What I'm going to do, is to delete some of them.

Let's start from network configuration. I don't know what initial OpenStack scripts do. Let's leave them off for now.
Basic requirements from Google:
DHCP note: https://github.com/Altoros/bosh/commit/cbc2a2938d0b19c88a93b23f7d770ed496341c28
I'm not sure where I should set a default MTU setting...
Anyway, it can be done in bash like this(assuming that eth0 is the target interface):
```
ifconfig eth0 mtu 1400
```

Now, we should disable IPv6. Basically, we need to add this into /etc/sysctl.conf:
```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

Repo with tool to build Debian for GCE:
https://github.com/camptocamp/build-debian-cloud
Has some interesting configuration scripts.

Also, there's a little problem. I don't really get, how to set NTP server.
It's all very odd for me. Looks like there should be separate files for that.

That's what I did with networking:
```
#!/usr/bin/env bash

set -e

base_dir=$(readlink -nf $(dirname $0)/../..)
source $base_dir/lib/prelude_apply.bash

# Remove persistent device names so that eth0 comes up as eth0
rm -rf $chroot/etc/udev/rules.d/70-persistent-net.rules

cat >> $chroot/etc/network/interfaces <<EOS
auto eth0
iface eth0 inet dhcp
EOS

# Set MTU recommended by Google
ifconfig eth0 mtu 1460

# Disable IPv6, as it is not supported by GCE
cat >> $chroot/etc/sysctl.conf <<EOS
# IPv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOS

# Hostname will be set by startup script, /usr/share/google/set-hostname
rm $chroot/etc/hostname

# Adding required entries to /etc/hosts
cat >> $chroot/etc/hosts <<EOS
169.254.169.254 metadata.google.internal metadata
EOS

# Disabling iptables by default
run_in_chroot $chroot "
service iptables save
service iptables stop
chkconfig iptables off
"
```

Well, big surprise. I should have read about this earlier https://github.com/camptocamp/build-debian-cloud/blob/master/gce
GCE specific scripts for setup.

1. Add Google repo to sources:
```
#!/bin/bash
# Basic sources.list. The current packages work on all supported versions of
# Debian, so at least for now we're using a distribution name independent of
# those. The repository is GPG signed, so temporarily import the key.
wget -O - https://goog-repo.appspot.com/debian/key/public.gpg.key | apt-key add -
cat > /etc/apt/sources.list.d/goog.list <<EOF
deb     http://goog-repo.appspot.com/debian pigeon main
EOF
```
2. Serial console output:
```
#!/bin/bash
# Prepare GRUB to install itself correctly.

# Set up a device map for GRUB to use while loopback-mounted:
echo "(hd0) ${base_device_path}" > $imagedir/boot/grub/device.map
echo "(hd0,1) ${device_path}" >> $imagedir/boot/grub/device.map

# Configure GRUB timeouts for fast booting:
sed -i -e '/GRUB_TIMEOUT=/s/^.*/GRUB_TIMEOUT=0/' $imagedir/etc/default/grub

# Direct GRUB and Linux console output to ttyS0 to facilitate gcutil
# getserialportoutput.
#
# The backreference on the next line avoids spurious spaces in the resulting
# kernel command line.
sed -i -e '/GRUB_CMDLINE_LINUX=/s/\( \)\?"$/\1console=ttyS0,38400n8"/' \
    $imagedir/etc/default/grub
sed -i -e '/GRUB_TERMINAL=/s/^.*/GRUB_TERMINAL="serial --unit=0 --speed=38400 --word=8 --parity=no --stop=1"' \
    $imagedir/etc/default/grub
```
3. Install required packages:
```
#!/bin/bash
# Add GCE startup scripts and packages to the image.

DEBS="google-compute-daemon google-startup-scripts image-bundle"
for deb in $DEBS; do
  apt-get install --force-yes -y ${deb}
done

4. Set hostname (optional?):
```
#!/bin/bash

# Set the hostname from DHCP.
chroot $imagedir ln -s /usr/share/google/set-hostname /etc/dhcp/dhclient-exit-hooks.d/set-hostname
```