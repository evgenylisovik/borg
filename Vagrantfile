# -*- mode: ruby -*-
# vi: set ft=ruby :
$scriptClient = <<-SCRIPTCLIENT
  apt install borgbackup -y
  cp /vagrant/client_ssh/id_ed25519.pub /home/vagrant/.ssh/
  cp /vagrant/client_ssh/id_ed25519 /home/vagrant/.ssh/
  cp /vagrant/client_ssh/id_ed25519.pub /root/.ssh/
  cp /vagrant/client_ssh/id_ed25519 /root/.ssh/
  ssh-keyscan -t ed25519 192.168.56.11 | cat >> /home/vagrant/.ssh/known_hosts
  ssh-keyscan -t ed25519 192.168.56.11 | cat >> /root/.ssh/known_hosts
  chown -R vagrant:vagrant /home/vagrant/
  chown 644 /root/.ssh/id_ed25519
  chown 644 /home/vagrant/.ssh/id_ed25519
  sudo -i
  BORG_PASSPHRASE='testpassphrase' borg init --encryption=repokey borg@192.168.56.11:/var/backup
  echo -e '[Unit]\nDescription=Borg Backup\n\n[Service]\nType=oneshot\n\nEnvironment="BORG_PASSPHRASE=testpassphrase"\nEnvironment=REPO=borg@192.168.56.11:/var/backup/\nEnvironment=BACKUP_TARGET=/etc\nExecStart=/bin/borg create \\\n   --stats             \\\n   ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}\nExecStart=/bin/borg check ${REPO}\nExecStart=/bin/borg prune \\\n     --keep-minutely 60        \\\n      --keep-hourly 24     \\\n    --keep-daily  90      \\\n    --keep-monthly 12     \\\n   --keep-yearly 1   \\\n${REPO}\nStandardOutput=syslog\nStandardError=syslog\nSyslogIdentifier=borg' >  /etc/systemd/system/borg-backup.service 
  echo -e '[Unit]\nDescription=Borg Backup\n\n[Timer]\nOnUnitActiveSec=5min\nUnit=borg-backup.service\n\n[Install]\nWantedBy=multi-user.target' >  /etc/systemd/system/borg-backup.timer
  systemctl enable rsyslog
  touch /var/log/borg
  touch /etc/rsyslog.d/borg.conf
  echo 'if \$programname == "borg" then /var/log/borg' | sudo tee /etc/rsyslog.d/borg.conf
  systemctl restart rsyslog
  touch /etc/logrotate.d/borg.conf
  echo -e '/var/log/borg {\n  hourly\n  rotate 3\n  size 2K\n  compress\n  delaycompress\n}' > /etc/logrotate.d/borg.conf
  systemctl daemon-reload
  systemctl enable borg-backup.service borg-backup.timer
  systemctl start borg-backup.service
  systemctl start borg-backup.timer  
  SCRIPTCLIENT

$script = <<-SCRIPT 
  apt install borgbackup -y
  adduser borg
  mkdir /home/borg/.ssh
  touch /home/borg/.ssh/authorized_keys
  chown -R borg:borg /home/borg/
  chmod 700 /home/borg/.ssh
  chmod 600 /home/borg/.ssh/authorized_keys
  cat /vagrant/client_ssh/id_ed25519.pub >> /home/borg/.ssh/authorized_keys
  mkdir /var/backup
  echo "y" | mkfs.ext4 /dev/sdb
  mount /dev/sdb /var/backup
  rm -r /var/backup/lost+found
  chown -R borg:borg /var/backup
  uuid=$(blkid | awk '$1=/sdb/ {print $2}' | cut -c 1-5,7-42)
  echo $uuid /var/backup   ext4     defaults   0 0 >> /etc/fstab  
  reboot  
  SCRIPT

  machines = [
    {
      :name => "backupServer",
      :eth0 => "192.168.56.11",
      :mem => "1024",
      :cpu => "2",
      :script => $script            
    },
    {
      :name => "client",
      :eth0 => "192.168.56.10",
      :mem => "1024",
      :cpu => "1",
      :script => $scriptClient         
    }
 
]

Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"
  config.vm.synced_folder "vagrant/", "/vagrant"
  config.vm.provider "virtualbox" do |vb|
    vb.customize ["modifyvm", :id, "--memory", "1024"]
  end
  machines.each do |options|
    config.vm.define options[:name] do |config|
      config.vm.hostname = options[:name]
      config.vm.network :private_network, ip: options[:eth0]  
      config.vm.provider "virtualbox" do |v|
        v.customize ["modifyvm", :id, "--name", options[:name]]
        v.customize ["modifyvm", :id, "--memory", options[:mem]]
        v.customize ["modifyvm", :id, "--cpus", options[:cpu]]       
        if config.vm.hostname == "backupServer" 
          v.customize ['createhd', '--filename', './sata7.vdi', '--variant', 'Fixed', '--size', 2000]
          #  v.customize ['storagectl', :id, '--name', 'SATA', '--add', 'sata' ]  Выходит ошибка: VBoxManage: error: Too many storage controllers of this type
          # т.к. в ubuntu уже есть контроллер sata. Лечится добавлением контроллера SAS.
          v.customize ["storagectl", :id, "--name", "SAS Controller", "--add", "sas", '--portcount', 2]
          v.customize ['storageattach', :id,  '--storagectl', 'SAS Controller', '--port', 0, '--device', 0, '--type', 'hdd', '--medium', './sata7.vdi']                               
        end                 
      end
      config.vm.provision "shell", inline: options[:script]
    end    
  end
end



  

