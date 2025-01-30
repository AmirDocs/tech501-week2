# Sparta App Deployment Guide

This guide provides step-by-step instructions to deploy the Sparta App on Azure using Virtual Machines (VMs), NGINX, MongoDB, and PM2. Follow these steps to set up and deploy the application.

---

## Table of Contents

1. [Prerequisites before undergoing task](#prerequisites)
2. [Step 1: Clone the Repository](#step-1-clone-the-repository)
3. [Step 2: Set Up Virtual Machines on Azure](#step-2-set-up-virtual-machines-on-azure)
   - [Create the Sparta App VM](#create-the-sparta-app-vm)
   - [Create the Database VM](#create-the-database-vm)
4. [Step 3: Install and Configure MongoDB on the Database VM](#step-3-install-and-configure-mongodb-on-the-database-vm)
5. [Step 4: Deploy the Application on the Sparta App VM](#step-4-deploy-the-application-on-the-sparta-app-vm)
6. [Step 5: Set Up NGINX as a Reverse Proxy](#step-5-set-up-nginx-as-a-reverse-proxy)
7. [Step 6: Manage the Application with PM2](#step-6-manage-the-application-with-pm2)
8. [Step 7: Create and Use VM Images](#step-7-create-and-use-vm-images)
9. [Troubleshooting](#troubleshooting)
10. [Conclusion](#conclusion)

---

## Prerequisites before undergoing task

Before starting, you will need the following

1. **An Azure Account**: Sign up for an [Azure account](https://azure.microsoft.com/).
2. **A GitHub Account**: Create a [GitHub account](https://github.com/).
3. **Your SSH Key Pair**: Generate an SSH key pair for secure access to your VMs. Use the following command:
   ```bash
   ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
* * * * *

Step 1: Clone the Repository
----------------------------

Clone the Sparta App repository to your local machine:

```bash
git clone https://github.com/AmirDocs/tech501-sparta-app.git
cd tech501-sparta-app
```
* * * * *

Step 2: Set Up Virtual Machines on Azure
----------------------------------------

### Create the Sparta App VM

-   **Name**: `tech501-amir-sparta-app-vm`

-   **OS**: Ubuntu Server 22.04 LTS

-   **Size**: Standard B1s

-   **Authentication**: SSH public key

-   **SSH Key Path**: `/home/amirbile/.ssh/tech501-amir-az-key`

-   **Public IP**: `20.26.126.25`

-   **Networking**:

    -   Subnet: Public subnet

    -   NSG: `tech501-amir-sparta-app-allow-HTTP-SSH-3000`

    -   Inbound Rules:

        -   Priority 300: Allow SSH (port 22)

        -   Priority 310: Allow HTTP (port 80)

        -   Priority 320: Allow custom port 3000 (for the app)

### Create the Database VM

-   **Name**: `tech501-amir-second-deploy-db-vm`

-   **OS**: Ubuntu Server 22.04 LTS

-   **Size**: Standard B1s

-   **Authentication**: SSH public key

-   **SSH Key Path**: `/home/amirbile/.ssh/tech501-amir-az-key`

-   **Public IP**: `4.234.1.14`

-   **Networking**:

    -   Subnet: Private subnet (restrict direct internet access)

    -   NSG Rules: Allow SSH (port 22) and MongoDB (port 27017)

* * * * *

Step 3: Install and Configure MongoDB on the Database VM
--------------------------------------------------------

1.  **SSH into the Database VM**:

    ```bash
    ssh -i /home/amirbile/.ssh/tech501-amir-az-key adminuser@4.234.1.14
    ```

2.  **Update and Install MongoDB**:

    ```bash
    sudo apt-get update -y
    sudo apt-get upgrade -y
    sudo apt-get install -y mongodb
    ```

3.  **Configure MongoDB**:

    -   Edit the MongoDB configuration file:

        ```bash
        sudo nano /etc/mongodb.conf
        ```

    -   Change `bind_ip` to `0.0.0.0` to allow remote connections:

        ```bash
        bind_ip = 0.0.0.0
        ```

    -   Save and exit (`CTRL + X`, then `Y`, then `Enter`).

4.  **Restart MongoDB**:

    ```bash
    sudo systemctl restart mongodb
    sudo systemctl status mongodb
    ```
* * * * *

Step 4: Deploy the Application on the Sparta App VM
---------------------------------------------------

1.  **SSH into the Sparta App VM**:

    ```bash
    ssh -i /home/amirbile/.ssh/tech501-amir-az-key adminuser@20.26.126.25
    ```

2.  **Clone the Repository**:

    ```bash    
    git clone https://github.com/AmirDocs/tech501-sparta-app.git
    cd tech501-sparta-app/repo/app
    ```

3.  **Install Dependencies**:

    ```bash
    npm install
    ```

4.  **Set the Database Connection**:\
    Export the MongoDB connection string:

    ```bash
    export DB_HOST="mongodb://<DB_VM_Private_IP>:27017/yourdatabase"
    ```

    Replace `<DB_VM_Private_IP>` with the private IP of the Database VM.

5.  **Start the Application**:

    ``bash
    npm start
    ``

6.  **Verify Deployment**:\
    Open a browser and navigate to your database VM's IP:

    http://20.26.126.25:3000

* * * * *

Step 5: Set Up NGINX as a Reverse Proxy
---------------------------------------

1.  **Install NGINX**:

    ```bash
    sudo apt-get install nginx -y
    ```

2.  **Configure NGINX**:

    -   Backup the default configuration:

        ```bash
        sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/default.backup
        ```

    -   Edit the configuration file:

        ```bash
        sudo nano /etc/nginx/sites-available/default
        ```

    -   Replace the `location /` block with:

    ```bash
        server {
        listen 80;
        server_name _;

        location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
       }
       }
    ```

3.  **Test and Reload NGINX**:

    ```bash
    sudo nginx -t
    sudo systemctl reload nginx
    ```

4.  **Verify NGINX Setup**:\
    Open a browser and navigate to:
    http://20.26.126.25

* * * * *

Step 6: Manage the Application with PM2
---------------------------------------

1.  **Install PM2**:

    ```bash
    sudo npm install -g pm2
    ```

2.  **Start the Application with PM2**:

    ```bash
    pm2 start app.js
    ```

3.  **Ensure PM2 Starts on Boot**:

    ```bash
    pm2 startup systemd
    pm2 save
    ```

4.  **Check Application Status**:

    ```bash
    pm2 status
    ```

* * * * *

Step 7: Create and Use VM Images
--------------------------------

1.  **Capture the Sparta App VM Image**:

    -   Navigate to the VM in the Azure Portal.

    -   Click "Capture" and name the image `tech501-amir-app-image`.

2.  **Deploy New VMs from the Image**:

    -   Use the captured image to create new VMs with identical configurations.

* * * * *

Troubleshooting
---------------

-   **Application Not Running**: Check logs with:

    ```bash
    pm2 logs app.js
    ```

-   **NGINX Errors**: Test the configuration with:

    ```bash
    sudo nginx -t
    ```

-   **Database Connection Issues**: Verify MongoDB is running and the connection string is correct.

* * * * *

Conclusion
----------

You have successfully deployed the Sparta App on Azure using VMs, NGINX, MongoDB, and PM2. This setup ensures scalability, reliability, and efficient management of your application.