# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'yaml'

base_dir = File.expand_path(File.dirname(__FILE__))
conf = YAML.load_file(File.join(base_dir, "vagrant.yml"))

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # if you want to use vagrant-cachier,
  # please install vagrant-cachier plugin.
  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.enable :apt
  end

  config.vm.box = "capgemini/mesos"
  config.vm.box_version = conf['mesos_version']

  ansible_groups = {
    "mesos-masters" => ["master1", "master2", "master3"],
    "mesos-slaves" => ["slave1"],
    "all:children" => ["mesos-masters", "mesos-slaves"]
  }

  # Mesos master nodes
  master_n = conf['master_n']
  master_infos = (1..master_n).map do |i|
    node = {
      :zookeeper_id    => i,
      :quorum          => (master_n.to_f/2).ceil,
      :hostname        => "master#{i}",
      :ip              => conf['master_ipbase'] + "#{10+i}",
      :mem             => conf['master_mem'],
      :cpus            => conf['master_cpus'],
    }
  end
  zookeeper_url = "zk://"+master_infos.map{|master| master[:ip]+":2181"}.join(",")
  zookeeper_conf = master_infos.map{|master| "server.#{master[:zookeeper_id]}"+"="+master[:ip]+":2888:3888"}.join("\n")
  consul_join = master_infos.map{|master| master[:ip]}.join(" ")

  # Mesos slave nodes
  slave_n = conf['slave_n']
  slave_infos = (1..slave_n).map do |i|
    node = {
      :hostname => "slave#{i}",
      :ip => conf['slave_ipbase'] + "#{10+i}",
      :mem => conf['slave_mem'],
      :cpus => conf['slave_cpus'],
    }
  end

  master_infos.flatten.each_with_index do |node|
    config.vm.define node[:hostname] do |cfg|
      cfg.vm.provider :virtualbox do |vb, override|
        override.vm.hostname = node[:hostname]
        override.vm.network :private_network, :ip => node[:ip]
        override.vm.provision :hosts

        vb.name = 'vagrant-mesos-' + node[:hostname]
        vb.customize ["modifyvm", :id, "--memory", node[:mem], "--cpus", node[:cpus] ]

        override.vm.provision "ansible" do |ansible|
          ansible.playbook = "site.yml"
          ansible.sudo = true
          ansible.extra_vars = {
            zookeeper_id: node[:zookeeper_id],
            zookeeper_conf: zookeeper_conf,
            zookeeper_url: zookeeper_url,
            mesos_quorum: node[:quorum],
            consul_join: consul_join,
            consul_advertise: node[:ip],
            consul_bootstrap_expect: master_n,
            mesos_local_address: node[:ip],
            consul_bind_addr: node[:ip]
          }
          ansible.groups = ansible_groups
        end
      end
    end
  end

  slave_infos.flatten.each_with_index do |node|
    config.vm.define node[:hostname] do |cfg|
      cfg.vm.provider :virtualbox do |vb, override|
        override.vm.hostname = node[:hostname]
        override.vm.network :private_network, :ip => node[:ip]
        override.vm.provision :hosts

        vb.name = 'vagrant-mesos-' + node[:hostname]
        vb.customize ["modifyvm", :id, "--memory", node[:mem], "--cpus", node[:cpus] ]

        override.vm.provision "ansible" do |ansible|
          ansible.playbook = "site.yml"
          ansible.sudo = true
          ansible.extra_vars = {
            zookeeper_url: zookeeper_url,
            mesos_quorum: node[:quorum],
            consul_join: consul_join,
            consul_advertise: node[:ip],
            consul_is_server: false,
            consul_bootstrap_expect: master_n,
            mesos_local_address: node[:ip],
            consul_bind_addr: node[:ip]
          }
          ansible.groups = ansible_groups
        end
      end
    end
  end

  # If you want to use a custom `.dockercfg` file simply place it
  # in this directory.
  if File.exist?(".dockercfg")
    config.vm.provision :shell, :priviledged => true, :inline => <<-SCRIPT
      cp /vagrant/.dockercfg /root/.dockercfg
      chmod 600 /root/.dockercfg
      chown root /root/.dockercfg
    SCRIPT
  end
end