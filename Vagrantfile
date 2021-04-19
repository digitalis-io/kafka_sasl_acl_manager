########################################################################################################################
# Vagrant file for running virtual machines on the localhost, so you can test the ansible provisioning locally.
########################################################################################################################
# NOTE:
# - Vagrant + Ansible inventory synchronisation is highly awkward. Instead, most ansible plays must be outside of
# vagrant. We only run the Ansible for just the site.yml playbook. Vagrant basically creates the VMs, simulates DNS,
# sets up the SSH keys and sets up extra storage files to simulate the production hardware scenarios.
#----------------------------------------------------------------------------------------------------------------------#
# Sanity checks (try fail early and caution)
if ENV['VAGRANT_DNS'] && !Vagrant.has_plugin?('landrush')
    raise 'Missing required plugin \'landrush\', run `vagrant plugin install landrush`'
  end
   
  ########################################################################################################################
  # Storage Defaults for Boxes
  PRIMARY = { name: 'data', size: 10, serial: 'data', device: 'disk/by-id/data' }.freeze
  HOST_EXTRA = { name: 'host_extra', size: 15, serial: 'host_extra', device: 'disk/by-id/host_extra' }.freeze 

  ########################################################################################################################
  # Host VM size and storage per group the ansible inventory targets
  hosts = [
    { name: 'loglh1v', memory: 1536, storage: [HOST_EXTRA, PRIMARY] },
    { name: 'loglh2v', memory: 1536, storage: [HOST_EXTRA, PRIMARY] },
    { name: 'loglh3v', memory: 1536, storage: [HOST_EXTRA, PRIMARY] }
  ]
   
  ########################################################################################################################
  # Include defaults for provider config and plugins
  ENV['VAGRANT_DNS'] ||= 'true'
  ENV['LANDRUSH_TLD'] ||= 'vagrant.test'
  ENV['BOX_SOURCE'] ||= 'external'
  # Where to place additional data storage if using virtualbox. Not needed for libvirt.
  ENV['VB_STORAGE_DIR'] ||= '/tmp'
  ENV['VB_CONTROLLER'] ||= 'SATA' # SATA Controller' The name may vary
   
  ########################################################################################################################
  def general(config)
    # Chose preference for providers
    case ENV['BOX_SOURCE']
    when 'internal'
    config.vm.box = "CentOS/RedHat/Ubuntu/Other"
    config.vm.box_url = "custom internal box repo URL"
    config.vm.box_version = "1.0"
    when 'external'
    config.vm.box = 'ubuntu/focal64'
    else
    raise "env var BOX_SOURCE=\"#{ENV['BOX_SOURCE']}\" invalid. Only 'internal' or 'external' expected."
    end
    # skip box update check?
    config.vm.box_check_update = ENV['UPDATE_BOX']
  end
   
  ########################################################################################################################
  # Support DNS
  # For OSX: host_redirect_dns is enabled by default so set sudo rules to allow
  # vagrant to update /etc/resolver
  # For Linux with NetworkManager: disable host_redirect_dns for Linux if dnsmasq
  # is run by NetworkManager.
  # - Default of true installs a separate dnsmasq and cause conflicts/problems
  # with NetworkManager's built in dnsmasq
  # - Instead, manually setup with `echo 'server=/vagrant.test/127.0.0.1#10053' > /etc/NetworkManager/dnsmasq.d/vagrant-landrush`
  def landrush(config)
    return unless ENV['VAGRANT_DNS']
    config.landrush.enabled = ENV['VAGRANT_DNS']
    config.landrush.tld = ENV['LANDRUSH_TLD']
    return unless Vagrant::Util::Platform.linux?
    config.landrush.guest_redirect_dns = false
    config.landrush.host_redirect_dns = false
  end
   
  ########################################################################################################################
  Vagrant.configure(2) do |config|
    general config
    landrush config
    config.vbguest.auto_update = false
   
    hosts.each do |h|
    config.vm.define "#{h[:name]}.#{ENV['LANDRUSH_TLD']}" do |v|
    v.vm.hostname = "#{h[:name]}.#{ENV['LANDRUSH_TLD']}"
    v.vm.provider :libvirt do |domain, _|
    domain.memory = h[:memory]
    domain.cpus = 2
    domain.disk_bus = 'virtio'
    domain.nic_model_type = 'virtio'
    h[:storage].each do |s|
    domain.storage :file, :allow_existing => false, :serial => "#{s[:serial]}", :size => "#{s[:size]}G"
    end
    end
    v.vm.provider :virtualbox do |vb, _|
    vb.memory = h[:memory]
    vb.cpus = 2
    vb.customize ['modifyvm', :id, '--natdnshostresolver1', 'on']
    end
    end
    end
   
    # manually replace insecure vagrant key with your own SSH key for vagrant user
    pub_key = File.read("#{ENV['HOME']}/.ssh/id_rsa.pub").chomp
    config.vm.provision 'SSH Keys', type: 'shell' do |s|
    s.inline = "if ! grep '#{pub_key}' /home/vagrant/.ssh/authorized_keys; then echo '#{pub_key}' >> /home/vagrant/.ssh/authorized_keys; fi"
    end
   
   
    # Use the local ansible provisioner instead, certain tasks require that the cluster is up and functioning in order
    # to work.
    config.vm.provision 'ansible' do |ansible|
    ansible.playbook = 'site.yml'
    ansible.inventory_path = 'inventories/hosts.yml'
    end
  end
   