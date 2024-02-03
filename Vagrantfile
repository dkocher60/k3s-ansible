# ENV['VAGRANT_NO_PARALLEL'] = 'no'
# NODE_ROLES = ["server-0", "server-1", "server-2", "agent-0", "agent-1"]
NODE_ROLES = ["server-0",  "agent-0", "agent-1"]
NODE_BOXES = ["gyptazy/ubuntu22.04-arm64", "gyptazy/ubuntu22.04-arm64", "gyptazy/ubuntu22.04-arm64"]
# NODE_BOXES = ['gyptazy/ubuntu22.04-arm64', 'gyptazy/ubuntu22.04-arm64', 'gyptazy/ubuntu22.04-arm64', 'gyptazy/ubuntu22.04-arm64']
# NODE_BOXES = ['aphorise/debian12-arm64', 'aphorise/debian12-arm64', 'aphorise/debian12-arm64', 'aphorise/debian12-arm64']
NODE_CPUS = 2
NODE_MEMORY = 2048
# Virtualbox >= 6.1.28 require `/etc/vbox/network.conf` for expanded private networks
NETWORK_PREFIX = "10.10.10"

def provision(vm, role, node_num)
  vm.box = NODE_BOXES[node_num]
  vm.hostname = role
  # We use a private network because the default IPs are dynamically assigned
  # during provisioning. This makes it impossible to know the server-0 IP when
  # provisioning subsequent servers and agents. A private network allows us to
  # assign static IPs to each node, and thus provide a known IP for the API endpoint.
  node_ip = "#{NETWORK_PREFIX}.#{100+node_num}"
  # An expanded netmask is required to allow VM<-->VM communication, virtualbox defaults to /32
  vm.network "private_network", ip: node_ip, netmask: "255.255.255.0"

  vm.provision "ansible", run: 'once' do |ansible|
    ansible.compatibility_mode = "2.0"
    ansible.playbook = "playbook/site.yml"
    ansible.groups = {
      "server" => NODE_ROLES.grep(/^server/),
      "agent" => NODE_ROLES.grep(/^agent/),
      "k3s_cluster:children" => ["server", "agent"],
    }
    ansible.extra_vars = {
      k3s_version: "v1.29.0+k3s1",
      api_endpoint: "#{NETWORK_PREFIX}.100",
      token: "myvagrant",
      # Required to use the private network configured above
      extra_server_args: "--node-external-ip #{node_ip} --flannel-iface ens256",
      extra_agent_args: "--node-external-ip #{node_ip} --flannel-iface ens256",
      # Optional, left as reference for ruby-ansible syntax
      # extra_service_envs: [ "NO_PROXY='localhost'" ],
      # server_config_yaml: <<~YAML
      #   write-kubeconfig-mode: 644
      #   kube-apiserver-arg:
      #     - advertise-port=1234
      # YAML
    }
  end
end

Vagrant.configure("2") do |config|
  config.vm.provider "vmware_desktop" do |v|
    v.cpus                              = NODE_CPUS
    v.memory                            = NODE_MEMORY
    v.linked_clone                      = true
    v.allowlist_verified                = true
  end
  config.ssh.keep_alive = true

  NODE_ROLES.each_with_index do |name, i|
    config.vm.define name do |node|
      provision(node.vm, name, i)
    end
  end

end
