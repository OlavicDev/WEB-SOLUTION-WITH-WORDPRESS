# WEB-SOLUTION-WITH-WORDPRESS
As a DevOps engineer, setting up a web solution with WordPress involves several key steps that integrate development, operations, and system administration principles to ensure a robust, scalable, and maintainable web environment. Here's an introduction to this process:

## Overview
WordPress is one of the most popular content management systems (CMS) used for creating websites and blogs. It's known for its ease of use, extensive theme and plugin ecosystem, and strong community support. Deploying WordPress in a production environment involves several components and steps, which are crucial for ensuring performance, security, and scalability.

## Key Components
1. Web Server: Typically, Nginx or Apache is used to serve WordPress content.
2. Database Server: MySQL or MariaDB is commonly used for storing WordPress data.
3. PHP Runtime: PHP is required to run WordPress scripts.
4. Reverse Proxy: Nginx or Traefik can be used to handle SSL termination and load balancing.
5. Containerization: Docker can be used to package WordPress and its dependencies, making deployment more consistent across different environments.
6. Orchestration: Tools like Docker Compose or Kubernetes can manage multi-container WordPress deployments.
7. CI/CD Pipeline: Continuous Integration/Continuous Deployment pipelines automate the deployment process, ensuring that changes are tested and deployed efficiently.

# Deploying a Websolution with Wordpress
With this project one will understand the concept of Three-tier Architecture
![image](https://github.com/user-attachments/assets/946a456c-8c54-4112-bf47-adecb5199a94)
In this case the:
laptop serves as the client(presentation tier)
Ec2 (where the wordpress will be installed) will serve as the web sever(application tier)
another Ec2 as the database(data tier)

## Step-1 Prepare a Web Server
1. Launch a RedHat EC2 instance that serve as Web Server. Create 3 volumes in the same AZ as the web server ec2 each of 10GB and attache all 3 volumes one by one to the web server.
Instance detail Instance detail.

![image](https://github.com/user-attachments/assets/2c6f6b93-da42-4ec8-b408-afe4577129df)
![image](https://github.com/user-attachments/assets/a5baa63c-6b07-4568-8323-1f4d4e1b475e)

2. open the linux terminal
![image](https://github.com/user-attachments/assets/3c604425-6905-4254-bb17-a100e5e6c8e4)

3. Use `lsblk` to inspect what block devices are attached to the server. All devices in Linux reside in /dev/ directory. Inspect with ls /dev/ and ensure all 3 newly created devices are there. Their name will likely be xvdf, xvdg and xvdh.
```
lsblk
```
![image](https://github.com/user-attachments/assets/ce74d95b-6f75-4de4-88d6-afd1e35d3de2)


4. Use `df -h` to see all mounts and free space on the server.
```
df -h
```
5a. Use gdisk utility to create a single partition on each of the 3 disks.
```
sudo gdisk /dev/xvdf
```
![image](https://github.com/user-attachments/assets/87a11156-ea40-46c9-999f-f75df91f9a30)

partition

Repeat the process for the other two disks
5b. Use `lsblk` utility to view the newly configured partitions on each of the 3 disks
```
lsblk
```
![image](https://github.com/user-attachments/assets/f27f717c-6978-45b7-86f7-ed6df1eca751)

6. Install `lvm` package
```
sudo yum install lvm2 -y
```

7. Use `pvcreate` utility to mark each of the 3 dicks as physical volumes (PVs) to be used by LVM. Verify that each of the volumes have been created successfully.
```
sudo pvcreate /dev/xvdf1 /dev/xvdg1 /dev/xvdh1

sudo pvs
```
![image](https://github.com/user-attachments/assets/65d2a57d-2425-48bd-a937-02ea9f9db771)

8. Use `vgcreate` utility to add all 3 PVs to a volume group (VG). Name the VG webdata-vg. Verify that the VG has been created successfully
```
sudo vgcreate webdata-vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1

sudo vgs
```
![image](https://github.com/user-attachments/assets/3c6510a2-7015-4e33-acaa-718983896ae5)

9. Use lvcreate utility to create 2 logical volume, apps-lv (Use half of the PV size), and logs-lv (Use the remaining space of the PV size). Verify that the logical volumes have been created successfully.

Note: apps-lv is used to store data for the Website while logs-lv is used to store data for logs.
```
sudo lvcreate -n apps-lv -L 14G webdata-vg

sudo lvcreate -n logs-lv -L 14G webdata-vg

sudo lvs
```
![image](https://github.com/user-attachments/assets/6c72c976-c34e-4b98-aa1e-7a79504e954f)

10. Verify the entire setup
```
sudo vgdisplay -v   #view complete setup, VG, PV and LV

lsblk
```

![image](https://github.com/user-attachments/assets/26cbf104-a8a6-4658-9cf5-3e0a86c7f9f8)

Use mkfs.ext4 to format the logical volumes with ext4 filesystem
```
sudo mkfs.ext4 /dev/webdata-vg/apps-lv

sudo mkfs.ext4 /dev/webdata-vg/logs-lv
```

11. Create /var/www/html directory to store website files and /home/recovery/logs to store backup of log data
```
sudo mkdir -p /var/www/html

sudo mkdir -p /home/recovery/logs
```
Mount `/var/www/html` on `apps-lv` logical volume
```
sudo mount /dev/webdata-vg/apps-lv /var/www/html
```

12. Use rsync utility to backup all the files in the log directory `/var/log` into `/home/recovery/logs` (This is required before mounting the file system)
```
sudo rsync -av /var/log /home/recovery/logs
```

![image](https://github.com/user-attachments/assets/9941bbe3-e4db-4757-8fa9-7ea00b901d81)

13. Mount `/var/log` on `logs-lv` logical volume (All existing data on /var/log is deleted with this mount process which was why the data was backed up)
```
sudo mount /dev/webdata-vg/logs-lv /var/log
```

14. Restore log file back into /var/log directory
```
sudo rsync -av /home/recovery/logs/log/ /var/log
```
15. Update /etc/fstab file so that the mount configuration will persist after restart of the server

Get the UUID of the device and Update the /etc/fstab file with the format shown inside the file using the UUID. Remember to remove the leading and ending quotes.
sudo blkid   # To fetch the UUID
```
sudo vi /etc/fstab
```
![image](https://github.com/user-attachments/assets/deb76535-e4bd-45b2-bf79-6c3c8e3d5780)

16. Test the configuration and reload daemon. Verify the setup
```
sudo mount -a  

sudo systemctl daemon-reload

df -h  
```

### Step 2 - Prepare the Database Server

1. **Launch and Attach Volumes**:
   Launch a second RedHat EC2 instance to serve as the DB Server. Repeat the initial setup steps from the Web Server, but instead of `apps-lv`, create `db-lv` and mount it to the `/db` directory. Create three 10GB volumes in the same AZ as the DB Server EC2 instance and attach all three volumes to the DB Server.

2. **Connect to the Instance**:
   Open the Linux terminal to begin configuration.
   ```sh
   ssh -i "ec2key.pem" ec2-user@54.219.132.188
   ```

3. **Inspect Block Devices**:
   Use `lsblk` to inspect the block devices attached to the server. Their names will likely be `xvdf`, `xvdk`, and `xvdh`.
   ```sh
   lsblk
   ```

4. **Create Partitions**:
   Use the `gdisk` utility to create a single partition on each of the three disks.
   ```
   sudo gdisk /dev/xvdf
   sudo gdisk /dev/xvdk
   sudo gdisk /dev/xvdh
   ```
   ![image](https://github.com/user-attachments/assets/4267a54f-c6a4-444e-876e-2a4cff20a4d4)


5. **View Partitions**:
   Use `lsblk` to view the newly configured partitions on each of the three disks.
   ```
   lsblk

   ```

6. **Install LVM**:
   Install the `lvm2` package.
   ```
   sudo yum install lvm2 -y
   ```

7. **Configure Physical Volumes and Volume Group**:
   Use `pvcreate` to mark each of the three disks as physical volumes (PVs) for LVM. Then, use `vgcreate` to add all three PVs to a volume group (VG) named `database-vg`. Verify the creation of the PVs and VG.
   ```sh
   sudo pvcreate /dev/xvdf1 /dev/xvdk1 /dev/xvdh1
   sudo pvs
   sudo vgcreate database-vg /dev/xvdf1 /dev/xvdk1 /dev/xvdh1
   sudo vgs
   ```
   ![image](https://github.com/user-attachments/assets/9ed55551-0d9f-4dc2-b03a-84afbb97f9a0)


8. **Create Logical Volume**:
   Use `lvcreate` to create a logical volume named `db-lv` with 20GB of the PV size. Verify the creation of the logical volume.
   ```
   sudo lvcreate -n db-lv -L 20G database-vg
   sudo lvs
   ```

9. **Format and Mount Logical Volume**:
   Use `mkfs.ext4` to format the logical volume with the `ext4` filesystem, then mount it to `/db`.
   ```
   sudo mkfs.ext4 /dev/database-vg/db-lv
   sudo mount /dev/database-vg/db-lv /db
   ```
   ![image](https://github.com/user-attachments/assets/aa7fb004-86aa-4b24-9526-add79018dbc2)


10. **Update /etc/fstab**:
    Update the `/etc/fstab` file to ensure the mount configuration persists after a restart.

    - Get the UUID of the device:
      ```
      sudo blkid
      ```
    
    - Update the `/etc/fstab` file:
      ```
      sudo vi /etc/fstab
      ```

      Add the following line (replace `your-uuid` with the actual UUID):
      ```
      UUID=your-uuid /db ext4 defaults 0 0
      ```
      ![image](https://github.com/user-attachments/assets/17b927a6-6dca-45cd-a403-2d30950a5a2f)


11. **Test and Verify Configuration**:
    Test the configuration and reload the daemon. Verify the setup.
    ```
    sudo mount -a   # Test the configuration
    sudo systemctl daemon-reload
    df -h   # Verify the setup
    ```
    ![image](https://github.com/user-attachments/assets/5843fed5-671e-4b1d-ba8a-0d2cc3481599)
