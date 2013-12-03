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

Right now, the process looks like this:

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
```

And finally - StemcellBuilder#build_stemcell:
```ruby
def build_stemcell
  gem_components = GemComponents.new(@build_number)
  gem_components.build_release_gems

  @stemcell_path = stemcell_builder_command.build

  File.exist?(@stemcell_path) || raise("#{@stemcell_path} does not exist")

  @stemcell_path
end
```

That leads us to BuilderCommand#build (and constructor):
```ruby
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
```
