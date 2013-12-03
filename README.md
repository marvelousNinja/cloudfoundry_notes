cloudfoundry_notes
==================

sudo mkdir /mnt/stemcells  
sudo mount --bind ~/tmp/stemcells /mnt/stemcells

rake release:create_dev_release
rake ci:build_stemcell[openstack,ubuntu,ruby]

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