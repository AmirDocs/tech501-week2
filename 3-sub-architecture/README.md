# Database Security Setup for Amir

## Overview

This guide outlines the steps to secure your database using a **3-subnet architecture** in Azure. The setup includes:
- A **public subnet** for the application VM.
- A **DMZ subnet** for the Network Virtual Appliance (NVA).
- A **private subnet** for the database VM.

The architecture ensures:
- Only the application VM has internet access.
- The database VM is isolated and accessible only through the NVA.
- Strict security rules are applied to minimize exposure.

---

## Key Steps

1. **Set Up the Virtual Network and Subnets**
2. **Deploy the Database VM**
3. **Deploy the Application VM**
4. **Test Connectivity Between VMs**
5. **Deploy the NVA VM**
6. **Configure Route Tables**
7. **Enable IP Forwarding on the NVA**
8. **Set Up IP Table Rules**
9. **Apply Stricter Security Rules for the Database VM**

---

## Security Principles

- **Minimize Exposure**: Close all unnecessary ports.
- **Restrict Access**: Only allow HTTP (Port 80) and SSH (Port 22) with strict rules.
- **Isolate the Database**: Ensure the database VM is not directly accessible from the internet.
- **Use an NVA**: Route all traffic through the NVA for added security.

---

## Architecture Diagram

- **Public Subnet**: Hosts the application VM with internet access.
- **DMZ Subnet**: Hosts the NVA VM for routing and firewall functionality.
- **Private Subnet**: Hosts the database VM with no public IP.

---

## Step 1: Create the Virtual Network and Subnets

### Configuration Details
- **Virtual Network Name**: amir-vnet
- **Address Space**: 10.1.0.0/16
- **Subnets**:
  - **public-subnet**: 10.1.1.0/24 (for the application VM)
  - **dmz-subnet**: 10.1.2.0/24 (for the NVA VM)
  - **private-subnet**: 10.1.3.0/24 (for the database VM, private access only)

### Azure CLI Command
```
az network vnet create\
  --name amir-vnet\
  --resource-group amir-resource-group\
  --address-prefixes 10.1.0.0/16\
  --subnet-name public-subnet\
  --subnet-prefix 10.1.1.0/24

az network vnet subnet create\
  --name dmz-subnet\
  --vnet-name amir-vnet\
  --resource-group amir-resource-group\
  --address-prefixes 10.1.2.0/24

az network vnet subnet create\
  --name private-subnet\
  --vnet-name amir-vnet\
  --resource-group amir-resource-group\
  --address-prefixes 10.1.3.0/24
```
* * * * *

Step 2: Deploy the Database VM
------------------------------

### Configuration Details

-   **VM Name**: tech501-amir-db-vm

-   **Subnet**: private-subnet

-   **No Public IP**

-   **SSH Key**: Use your existing SSH key for access.


### Azure CLI Command
```
az vm create\
  --name amir-db-vm\
  --resource-group amir-resource-group\
  --image Ubuntu2204\
  --vnet-name amir-vnet\
  --subnet private-subnet\
  --admin-username amir\
  --ssh-key-value ~/.ssh/id_rsa.pub\
  --public-ip-address ""
```
* * * * *

Step 3: Deploy the Application VM
---------------------------------

### Configuration Details

-   **VM Name**: amir-app-vm

-   **Subnet**: public-subnet

-   **Public IP**: Enabled

-   **Allow HTTP (Port 80) and SSH (Port 22)**

### Azure CLI Command

```
az vm create\
  --name amir-app-vm\
  --resource-group amir-resource-group\
  --image Ubuntu2204\
  --vnet-name amir-vnet\
  --subnet public-subnet\
  --admin-username amir\
  --ssh-key-value ~/.ssh/id_rsa.pub\
  --public-ip-address ""\
  --nsg-rule HTTP
```

### User Data Script

Add the following script to configure the application:

```
#!/bin/bash
cd /repo/nodejs20-sparta-test-app/app
export DB_HOST=mongodb://10.1.3.4:27017/posts
pm2 start app.js
```

* * * * *

Step 4: Test Connectivity Between VMs
-------------------------------------

### Ping the Database VM from the Application VM

```
ping 10.1.3.4
```

**Expected Output**:
```
64 bytes from 10.1.3.4: icmp_seq=1 ttl=64 time=0.5 ms
```
* * * * *

Step 5: Deploy the NVA VM
-------------------------

### Configuration Details

-   **VM Name**: amir-nva-vm

-   **Subnet**: dmz-subnet

-   **Temporary Public IP**: Enabled for setup

-   **Image**: Ubuntu Server 22.04 LTS

### Azure CLI Command

```
az vm create\
  --name amir-nva-vm\
  --resource-group amir-resource-group\
  --image Ubuntu2204\
  --vnet-name amir-vnet\
  --subnet dmz-subnet\
  --admin-username amir\
  --ssh-key-value ~/.ssh/id_rsa.pub\
  --public-ip-address ""
  ```

* * * * *

Step 6: Configure Route Tables
------------------------------

### Create a Route Table

```
az network route-table create\
  --name amir-route-table\
  --resource-group amir-resource-group\
  --location westuk ## correct this#
```

### Add Routes to the Route Table

```
az network route-table route create\
  --name to-private-subnet-route\
  --resource-group amir-resource-group\
  --route-table-name amir-route-table\
  --address-prefix 10.1.3.0/24\
  --next-hop-type VirtualAppliance\
  --next-hop-ip-address 10.1.2.4
  ```

### Associate Subnets with the Route Table

```
az network vnet subnet update\
  --name private-subnet\
  --vnet-name amir-vnet\
  --resource-group amir-resource-group\
  --route-table amir-route-table
  ```

* * * * *

Step 7: Enable IP Forwarding on the NVA
---------------------------------------

### Enable IP Forwarding in Azure

```
az network nic update\
  --name amir-nva-nic\
  --resource-group amir-resource-group\
  --ip-forwarding true
```

### Enable IP Forwarding on the NVA VM (Linux)

SSH into the NVA VM and run:

```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

* * * * *

Step 8: Set Up IP Table Rules
-----------------------------

### Configure IP Tables on the NVA VM

SSH into the NVA VM and run:

```
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
```

* * * * *

Step 9: Apply Stricter Security Rules for the Database VM
---------------------------------------------------------

### Restrict SSH Access

-   Allow SSH only from trusted IP addresses.

-   Update the Network Security Group (NSG) rules in Azure to enforce this.

### Example NSG Rule

```
az network nsg rule create\
  --name allow-ssh-trusted\
  --nsg-name amir-db-nsg\
  --resource-group amir-resource-group\
  --priority 100\
  --direction Inbound\
  --access Allow\
  --protocol Tcp\
  --source-address-prefixes <TRUSTED_IP>\
  --source-port-ranges '*'\
  --destination-address-prefixes '*'\
  --destination-port-ranges 22
  ```

* * * * *

Conclusion
----------

By following this guide, you have successfully set up a secure 3-subnet architecture in Azure. The database VM is now isolated, and all traffic is routed through the NVA for added security. Make sure to regularly review and update your security rules to maintain a robust defense against potential threats.