sudo groupadd -f k3s-admins
sudo usermod -aG k3s-admins $USER

Edit `/etc/rancher/k3s/config.yaml` and add these lines. If the file already exists, merge them into it instead of replacing the whole file.

write-kubeconfig-mode: "0640"
write-kubeconfig-group: "k3s-admins"



sudo systemctl restart k3s

newgrp k3s-admins