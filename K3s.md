sudo groupadd -f k3s-admins
sudo usermod -aG k3s-admins $USER

Edit `/etc/rancher/k3s/config.yaml` and add these lines. If the file already exists, merge them into it instead of replacing the whole file.

write-kubeconfig-mode: "0640"
write-kubeconfig-group: "k3s-admins"



sudo systemctl restart k3s

newgrp k3s-admins


  

## 7. Source of Truth Matrix

  

| Data / capability            | System of record | Read by / replicated to          | Notes                                      |
| ---------------------------- | ---------------- | -------------------------------- | ------------------------------------------ |
| ecommerce data entities      | Directus         | Medusa                           | entities are: regions, currencies, markets |
| Product family structure     | Directus         | Medusa, Storefront               | Canonical authoring lives in Directus      |
| Variant definitions and SKUs | Directus         | Medusa, Storefront               | SKU is immutable                           |
| Media assets and alt text    | Directus         | Storefront, optional Medusa refs | Directus DAM is authoritative              |
| Base prices                  | Directus         | Medusa                           | Upstream authored prices                   |



