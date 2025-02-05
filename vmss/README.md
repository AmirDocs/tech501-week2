# Azure Virtual Machine Scale Sets (VMSS) Deployment Guide

## Overview
Azure Virtual Machine Scale Sets (VMSS) allow you to deploy and manage a set of identical, load-balanced virtual machines (VMs). This ensures **high availability**, **scalability**, and **automated load distribution** across instances. VMSS is ideal for applications that need to handle varying workloads while maintaining performance and reliability.

---

## Why Use VMSS?
- **High Availability**: Distributes traffic across multiple instances to ensure your application remains available even if one or more instances fail.
- **Scalability**: Automatically scales the number of VM instances up or down based on demand, ensuring optimal resource utilization.
- **Load Balancing**: Integrates seamlessly with Azure Load Balancer or Azure Application Gateway to distribute traffic evenly across instances.
- **Cost Efficiency**: Pay only for the compute resources you use, with the ability to scale down during low-demand periods.
- **Simplified Management**: Easily manage, update, and configure multiple VM instances as a single entity.

---

## Prerequisites
Before proceeding with the deployment, ensure you have the following:
1. **Azure Subscription**: An active Azure account with sufficient permissions to create and manage resources.
2. **Azure CLI**: The Azure Command-Line Interface (CLI) installed on your local machine. You can install it from [here](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).

---

## Creating a Monitoring and Alerts Dashboard

- Go to the overview page of the VM.
- Monitoring tab.
  - You can see the graphs by default through Azure Monitor (similar to CloudWatch on AWS).
  - We want to turn these charts into a dashboard.

### Steps to Create the Dashboard
1. In the monitoring tab for the VM, scroll down to where you can see the charts.
2. Click the pin on the CPU average graph:
   - Create new.
   - Shared type.
   - Name: `amirb-shared-app-dashboard`
   - Click create and pin.
3. Add the network total metric and the disk operations to the dashboard you just created.

### To View the Dashboard
1. Go to `Dashboard hub`.
2. Click on your dashboard.
3. Click `Go to dashboard` at the top.

### Customizing the Dashboard
- You can change the size and layout of the dashboard by clicking Edit. Press Edit and then `x` on the tile popup.
- If you click on a metric, you can change the time frame and then click **Save to Dashboard** (might not work if you change the time frame without clicking the metric).

---

# Installing Apache Bench (Load Testing)

To install and run Apache Bench for load testing:

```bash
sudo apt-get install apache2-utils
```

Run a test with 1000 requests and 100 concurrent requests:

```bash
ab -n 1000 -c 100 http://yourwebsite.com/
ab -n 1000 -c 100 http://172.187.145.27/
```

---

# Auto Scaling

## Why Use Auto Scaling?

From worst to best:
1. **Fall over** (worst option).
2. **Dashboard monitoring**.
3. **Alerts**.
4. **Auto scaling** (best option).

### Types of Scaling
1. **Horizontal scaling** (scaling in or out by adding/removing instances).
2. **Vertical scaling** (scaling up or down by increasing/decreasing CPU, RAM, etc.).

### Azure Virtual Machine Scale Sets (VMSS)

- Similar to AWS Auto Scaling Groups.
- Ensures **high availability and scalability**.
- Custom autoscale settings:
  - Scale out when **CPU usage exceeds 75%**.
  - Start with **2 VMs** for **disaster recovery, redundancy, and high availability**.
  - Minimum VMs: **2**
  - Default VMs: **2**
  - Maximum VMs: **3**
  - Distribute VMs across different **availability zones** for redundancy:
    - Zone 1, 2, 3.
- The VMSS creates VMs from the provided **image**.
- Traffic is handled by a **load balancer** that distributes traffic across VMs.

#### Considerations
- If a VM image is created with **user data**, and you use that image for VMSS:
  - Stopping and restarting the VM will result in an **unhealthy status** because the user data only runs at initialization.
  - **Reimaging** the VM will run the user data script again, resetting the VM to its initial state (all other changes will be lost).

---

## Creating a VMSS

1. **VMSS Name:** `amirb-sparta-app-vmss`
2. **Availability Zones:** Zone 1, 2, 3
3. **Orchestration Mode:** Uniform
4. **Security Type:** Standard
5. **Scaling Mode:** Autoscaling
   - Click **Edit** to configure.
   - **Min:** 2, **Max:** 3, **Default:** 2
   - Scale out when **CPU threshold exceeds 75%**.
   - Click **Save**.
6. **Image:** Select an image from **My Images**.
7. **SSH Key:** Select an existing SSH key on Azure.
8. **OS Disk Type:** Standard SSD.
9. **Public IP:** Disabled (VMs will be accessed through a load balancer).
10. **Load Balancer:** Create a new load balancer:
    - Name: `amirb-app-lb`
    - Configure ports:
      - VM 1: SSH through port **50000**
      - VM 2: SSH through port **5001**
11. **Enable:**
    - Health monitoring.
    - Automatic repairs.
    - User data (**paste the following script**):

```bash
#!/bin/bash
cd /repo/app
pm2 start app.js
```

### Verifying the Deployment
- Enter the **public IP of the VMSS** into a browser.
- The app page should load (it might take a few minutes).
- **If you restart a VM (stop/start), you need to reimage at least one VM for the app to work.**
- **Health status** is either `healthy` or `unhealthy` if the VM is running. If not running, the status is blank.

---

## SSH into the VM

- Since there is **no public IP**, and the private IP is not accessible from a different VNet, SSH must be done via the **load balancer**.
- Use the load balancer's **public IP**:

```bash
ssh -i ~/.ssh/amir-az-key -p 5000 adminuser@4.234.17.239/
```

- `-p 50000` specifies the port assigned to the VM in the VMSS settings.

---

## Deleting the VMSS

1. Go to **VMSS** → Click **Delete**.
2. The **load balancer** must be deleted separately:
   - Go to **Load Balancers**.
   - Click **Delete**.
