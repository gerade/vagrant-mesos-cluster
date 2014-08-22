require 'json'

VAGRANTFILE_API_VERSION = "2"
DEFAULT_CLUSTER = "mesos"

def servers_by_role(cluster, role)
  out = []
  count = 0

  cluster.each do |k,v|
    if v["role"] == role
      count += 1
      out << { "id" => count, "host" => v["ip"] }
    end
  end

  return out
end

base_dir = File.expand_path(File.dirname(__FILE__))
cluster = JSON.parse(IO.read(File.join(base_dir, "clusters", ENV['CLUSTER'] || DEFAULT_CLUSTER, "cluster.json")))
zk_servers = servers_by_role(cluster, "zk")
zk_uri = "zk://" + zk_servers.map{|x| x["host"] + ":2181"}.join(",")

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  cluster.each do |hostname, info|

    config.vm.define hostname do |cfg|

      cfg.vm.provider :virtualbox do |vb, override|
        override.vm.box = "trusty64"
        override.vm.box_url = "https://cloud-images.ubuntu.com/vagrant/trusty/current/trusty-server-cloudimg-amd64-vagrant-disk1.box"
        override.vm.network :private_network, ip: "#{info["ip"]}"
        override.vm.hostname = hostname

        vb.name = 'vagrant-mesos-' + hostname
        vb.customize ["modifyvm", :id, "--memory", info["config"]["mem"], "--cpus", info["config"]["cpus"], "--hwvirtex", "on" ]
      end

      # provision nodes with ansible
      cfg.vm.provision :ansible do |ansible|
        ansible.verbose = "v"

        if info["role"] == "zk" then
          ansible.playbook = "ansible/zookeeper.yml"
          ansible.extra_vars = {
            zookeeper_myid: zk_servers.select{|x| x["host"] == info["ip"]}.first["id"],
            zookeeper_servers: zk_servers
          }
        else
          ansible.playbook = "ansible/mesosphere.yml"
          ansible.extra_vars = {
            mesos_mode: "#{info["role"]}",
            mesos_ip: "#{info["ip"]}",
            mesos_zk: "#{zk_uri}/mesos"
          }

          case info["role"]
          when "master"
            ansible.extra_vars.merge!({
              mesos_options_master: {
                cluster: "vagrant-mesos-cluster",
                work_dir: "/var/run/mesos",
                quorum: (servers_by_role(cluster, "master").length.to_f/2).ceil
              }
            })
          when "slave"
            ansible.extra_vars.merge!({})
          when "marathon"
            ansible.extra_vars.merge!({
              marathon_zk: "#{zk_uri}/marathon"
            })
          end
        end

      end

    end

  end

end
