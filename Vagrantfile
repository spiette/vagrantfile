# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'
require 'fileutils'

# Files used to customise Vagrantfile behaviour
defaults_file = 'test/vagrant/defaults.yml'
boxes_file = 'test/vagrant/boxes.yml'
group_vars_file = 'test/vagrant/group_vars.yml'

if File.file?(defaults_file)
  begin
    defaults = YAML.load_file(defaults_file)
  rescue
    $stderr.puts "#{defaults_file} was not load properly"
    defaults = Hash.new
  end
else
    $stderr.puts "#{defaults_file} is not existing"
    defaults = Hash.new
end
if File.file?(boxes_file)
  begin
    boxes = YAML.load_file(boxes_file)
  rescue
    $stderr.puts "#{boxes_file} was not load properly"
    boxes = Array.new
  end
else
    $stderr.puts "Creating default boxes file: #{boxes_file}"
    boxes = [
      {
        :hostname => 'vagrant',
        :memory => '256',
        :cpus => '1',
        :networks => [
          :private_network => [ :ip => '10.1.2.3' ]
        ],
        :ansible_playbook => 'site.yml',
        :ansible_hostvars => {
          'ansible_become' => true,
        },
        :ansible_groups => [ 'all', 'dev' ],
        :shell =>  { :path => '/bin/true' },
        :bats => '/vagrant/test/integration/defaults/bats',
      }
    ]
    FileUtils.mkdir_p File.dirname(boxes_file)
    File.open(boxes_file, 'w') {|f| f.write boxes.to_yaml }
end
if File.file?(group_vars_file)
  begin
    group_vars = YAML.load_file(group_vars_file)
  rescue
    group_vars = Hash.new
  end
end

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://atlas.hashicorp.com/search.

  config.ssh.insert_key = false

  # Set default domain
  if defaults.has_key?(:domain)
    domain = defaults[:domain]
  else
    domain = 'localdomain'
  end
  default_domain = domain
  # Populate groups
  ansible_groups = Hash.new
  boxes.each do |opts|
    if opts.has_key?(:domain)
      domain = opts[:domain]
    else
      domain = default_domain
    end
    fqdn = opts[:hostname] + "." + domain
    if opts.has_key?(:ansible_groups)
      # first convert the list into group_name => host
      opts[:ansible_groups].each do |group|
        if ansible_groups[group].is_a?(Array)
          ansible_groups[group].push(fqdn)
        else
          ansible_groups[group] = [fqdn]
        end
      end
    end
  end
  # now add the *:vars groups
  if group_vars.is_a?(Hash)
    ansible_groups.merge!(group_vars)
  end

  config.hostmanager.enabled = true
  config.hostmanager.manage_host = false
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true
  config.hostmanager.fqdn_friendly = true

  boxes.each do |opts|
    fqdn = opts[:hostname] + "." + domain
    config.vm.define fqdn do |node|
      node.vm.hostname = opts[:hostname]
      node.hostmanager.aliases = [ fqdn ]
      node.hostmanager.domain_name = domain
      if opts.has_key?(:box)
        node.vm.box = opts[:box]
      else
        node.vm.box = "centos/7"
      end
        [ 'libvirt', 'virtualbox' ].each do |provider|
        node.vm.provider provider do |vm|
          vm.memory = opts[:memory]
          vm.cpus = opts[:cpus]
        end
      end
      # ansible
      if opts.has_key?(:ansible_playbook)
        config.vm.provision "ansible" do |ansible|
          ansible.playbook = opts[:ansible_playbook]
          if opts.has_key?(:ansible_verbose)
            ansible.verbose = opts[:ansible_verbose]
          end
          if opts.has_key?(:ansible_hostvars)
            ansible.host_vars = { fqdn => opts[:ansible_hostvars] }
          end
          ansible.groups = ansible_groups
        end
      end
      # network options
      if opts.has_key?(:networks) and opts[:networks].is_a?(Array)
        opts[:networks].each do |network|
          if network.is_a?(Hash)
            network_identifier = network.keys[0]
            network_options = network[network_identifier]
            if network_options.is_a?(Hash)
              network_options.keys.each do |k|
                unless k.class == Symbol
                  $stdout.puts "Set your options using symbols, :myopt"
                end
              end
              config.vm.network network_identifier, network_options
            else
              $stdout.write("network_options is not a Hash, it's rather a #{network_options.class}\n")
            end
          end
        end
      end
      # shell
      if opts.has_key?(:shell)
        # using path
        if opts[:shell].has_key?(:path)
          config.vm.provision "shell", path: opts[:shell][:path]
        end
        # using inline
        if opts[:shell].has_key?(:inline)
          config.vm.provision "shell", inline: opts[:shell][:inline]
        end
      end
      # bats
      if opts.has_key?(:bats)
        config.vm.provision "shell", inline: 'yum install -q -y bats'
        config.vm.provision "shell", inline: "/usr/bin/bats #{opts[:bats]}"
      end
      # serverspec
      if opts.has_key?(:serverspec) and opts.has_key?(:serverspec_path)
        if not File.file?('Gemfile')
          # gem installation
          gemfile = Array.new
          gemfile.push('source "https://rubygems.org"')
          gemfile.push('gem "serverspec"')
          gemfile.push('gem "json"')
          File.open('Gemfile', 'w') { |file| file.write(gemfile.join("\n")) }
          system('bundle', 'install', '--binstubs', '--path',  'vendor')
        end
        config.vm.provision "shell", inline: "./bin/serverspec #{opts[:serverspec]}"
      end
    end
  end
end
