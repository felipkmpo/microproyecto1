Vagrant.configure("2") do |config|
  # Configuración global
  config.vm.box = "ubuntu/bionic64"

  # Número de nodos de Consul
  num_nodes = 3

  (1..num_nodes).each do |i|
    config.vm.define "consul_node_#{i}" do |node|
      node.vm.hostname = "consul-node-#{i}"

      # Asignar IP estática dentro del rango 192.168.100.x
      node.vm.network "private_network", ip: "192.168.100.#{10 + i}"

      # Asignación de memoria y CPU
      node.vm.provider "virtualbox" do |vb|
        vb.memory = 512
        vb.cpus = 1
      end

      # Provisión del sistema
      node.vm.provision "shell", inline: <<-SHELL
        # Actualizar e instalar herramientas necesarias
        sudo apt-get update
        sudo apt-get install -y unzip curl

        # Instalar Consul
        wget https://releases.hashicorp.com/consul/1.16.0/consul_1.16.0_linux_amd64.zip
        unzip consul_1.16.0_linux_amd64.zip
        sudo mv consul /usr/local/bin/
        sudo mkdir -p /etc/consul.d
      SHELL

      if i == 1
        # Configuración y provisión específicas para el nodo 1
        node.vm.provision "shell", inline: <<-SHELL
          # Instalar HAProxy
          sudo apt-get install -y haproxy

          # Configurar HAProxy
          sudo tee /etc/haproxy/haproxy.cfg > /dev/null <<-EOF
          global
              log /dev/log    local0
              log /dev/log    local1 notice
              chroot /var/lib/haproxy
              stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
              stats timeout 30s
              user haproxy
              group haproxy
              daemon

          defaults
              log     global
              option  httplog
              option  dontlognull
              timeout connect 5000
              timeout client  50000
              timeout server  50000

          frontend http_front
              bind *:80
              mode http
              option  httplog
              default_backend http_back

          backend http_back
              mode http
              balance roundrobin
              server consul_node_2 192.168.100.12:3002 check
              server consul_node_3 192.168.100.13:3003 check

          listen stats
              bind *:8404
              mode http
              stats enable
              stats uri /stats
              stats refresh 10s
              stats auth admin:admin
          EOF

          # Reiniciar HAProxy para aplicar la configuración
          sudo systemctl restart haproxy
        SHELL
      else
        # Configuración de Consul y Node.js para los nodos 2 y 3
        node.vm.provision "shell", inline: <<-SHELL
          # Configurar Consul en modo servidor
          sudo tee /etc/consul.d/consul.json > /dev/null <<-EOF
          {
            "server": true,
            "bootstrap_expect": 3,
            "data_dir": "/tmp/consul",
            "client_addr": "0.0.0.0",
            "bind_addr": "192.168.100.#{10 + i}",
            "advertise_addr": "192.168.100.#{10 + i}",
            "retry_join": ["192.168.100.11", "192.168.100.12", "192.168.100.13"],
            "ui": true
          }
          EOF

          # Iniciar Consul en segundo plano
          nohup consul agent -server -config-dir /etc/consul.d > /vagrant/consul_#{i}.log 2>&1 &

          # Instalar Node.js y configurar servidor básico
          sudo apt-get install -y nodejs npm
          mkdir ~/node_app
          cd ~/node_app

          # Crear archivo de servidor básico en Node.js
          tee server.js > /dev/null <<-EOF
          const http = require('http');
          const hostname = '0.0.0.0';
          const port = 3001 + #{i};
          const server = http.createServer((req, res) => {
            res.statusCode = 200;
            res.setHeader('Content-Type', 'text/plain');
            res.end('Hello World from Node.js on consul_node_#{i}!\n');
          });
          server.listen(port, hostname, () => {
            console.log('Server running at http://' + hostname + ':' + port + '/');
          });
          EOF

          # Ejecutar servidor Node.js en segundo plano
          nohup node server.js > server.log 2>&1 &
        SHELL
      end
    end
  end
end
