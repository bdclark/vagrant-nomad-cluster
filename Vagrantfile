# Cluster will be size of ip_list
# First node will be Consul server and Nomad server/client
# All other nodes will be Nomad clients
IP_LIST = [
  '192.168.77.20',
  '192.168.77.21',
  '192.168.77.22'
]

CONSUL_VERSION = '0.9.3'
NOMAD_VERSION = '0.8.4'

SCRIPT = <<SCRIPT
set -e

hashi_install() {
  local -r binary="$1"
  local -r version="$2"
  (
    cd /tmp || return
    rm -f "${binary}" "${binary}.zip"
    curl -fsSL -o "${binary}.zip" "https://releases.hashicorp.com/${binary}/${version}/${binary}_${version}_linux_amd64.zip"
    unzip "${binary}.zip" > /dev/null
    sudo install "${binary}" "/usr/bin/${binary}"
    rm -f "${binary}" "/tmp/${binary}.zip"
  )
  sudo mkdir -p "/etc/${binary}.d"
  sudo mkdir -p "/var/lib/${binary}"
}

hashi_systemd_unit() {
  local -r binary="$1"
  shift
  sudo tee "/etc/systemd/system/${binary}.service" > /dev/null <<EOF
[Unit]
Description=${binary} agent
Requires=network-online.target
After=network-online.target
[Service]
Restart=on-failure
ExecStart=/usr/bin/${binary} agent $@
ExecReload=/bin/kill -HUP $MAINPID
[Install]
WantedBy=multi-user.target
EOF
  sudo systemctl daemon-reload
  sudo systemctl enable "${binary}.service"
  sudo systemctl restart "${binary}.service"
}

echo "Installing dependencies..."
sudo apt-get update -qq
sudo apt-get install --yes curl unzip vim

echo "Installing Consul and Nomad..."
hashi_install consul #{CONSUL_VERSION}
hashi_install nomad #{NOMAD_VERSION}

sudo tee "/etc/nomad.d/config.hcl" > /dev/null <<EOF
advertise {
  rpc  = "$1"
  serf = "$1"
}
EOF

if [ "$1" = "#{IP_LIST[0]}" ]; then
  hashi_systemd_unit consul -bind $1 -client 0.0.0.0 -data-dir /var/lib/consul -server -bootstrap-expect 1 -ui
  hashi_systemd_unit nomad -config /etc/nomad.d -data-dir /var/lib/nomad -server -client -bootstrap-expect 1
else
  hashi_systemd_unit consul -bind $1 -client 0.0.0.0 -data-dir /var/lib/consul -retry-join #{IP_LIST[0]}
  hashi_systemd_unit nomad -config /etc/nomad.d -data-dir /var/lib/nomad -client
fi
nomad -autocomplete-install 2>/dev/null || true
SCRIPT

Vagrant.configure('2') do |config|
  config.vm.box = 'bento/ubuntu-18.04'

  config.vm.provider 'virtualbox' do |vb|
    vb.memory = '1024'
  end

  IP_LIST.each_with_index do |ip, index|
    node = "node#{index+1}"
    config.vm.define node do |machine|
      machine.vm.hostname = node
      machine.vm.network 'private_network', ip: ip
      if index == 0
        machine.vm.network 'forwarded_port', guest: 8500, host: 8500, host_ip: '127.0.0.1'  #, guest_ip: '127.0.0.1'
        machine.vm.network 'forwarded_port', guest: 4646, host: 4646, host_ip: '127.0.0.1'  #, guest_ip: '127.0.0.1'
      end

      machine.vm.provision 'docker'
      machine.vm.provision 'shell', inline: SCRIPT, args: [ip], privileged: false
    end
  end
end
