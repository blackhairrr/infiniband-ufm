#UFM-HA Deployment with Failover Test

OS - Ubuntu 24.04

<img width="428" height="317" alt="image" src="https://github.com/user-attachments/assets/34b6f137-8480-4c82-8da9-03f09716dec5" />


### Pre-Deployments
1- Install pacemaker, pcs and drbd-utils
```bash
sudo apt install pcs pacemaker drbd-utils docker.io resource-agents-extra
```

2- Install infiniband driver
	Link source - https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/
	
	tar -xzf MLNX_OFED_LINUX-24.10-4.1.4.0-ubuntu24.04-x86_64.tgz
	 sudo apt install bzip2
	 sudo ./mlnxofedinstall --add-kernel-support

Docker mode
```bash
docker run -it --name=ufm_installer --rm \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /etc/systemd/system/:/etc/systemd_files/ \
-v /opt/ufm/files/:/installation/ufm_files/ \
-v /tmp/license_file/:/home/robust \
mellanox/ufm-enterprise:latest \
--install
```


<img width="660" height="299" alt="image" src="https://github.com/user-attachments/assets/91e77075-b451-415e-b910-ef0e301d4179" />



```
systemctl daemon-reload
systemctl start ufm-enterprise
```

*UFM HA can run on one side first before bring up the secondary node


### Preparing DRBD Disk
---------------------------------------------------------------------------------------------
1. Verify volume group
<img width="519" height="111" alt="image" src="https://github.com/user-attachments/assets/25ba2109-56ee-4a75-bd7f-180862d38671" />
<img width="376" height="76" alt="image" src="https://github.com/user-attachments/assets/7400e791-372f-4871-aa7f-c310b37813ea" />

2. Create new lvm for DRBD
<img width="580" height="53" alt="image" src="https://github.com/user-attachments/assets/91a490e8-db01-4c22-a07f-77cbd581bba3" />
 
3. Verify lvm
<img width="666" height="60" alt="image" src="https://github.com/user-attachments/assets/ddec5716-881e-44e2-858b-9a9797c80ad5" />
<img width="566" height="126" alt="image" src="https://github.com/user-attachments/assets/b44dde4a-f426-4ae5-8f42-ea268ff206a7" />
<img width="436" height="37" alt="image" src="https://github.com/user-attachments/assets/389029e4-1243-4987-ae26-877ac13c4645" />


4. The volume path for DRBD directory **/dev/ubuntu-vg/ufm-dir-lv**

### Download UFM HA
1. Download the UFM HA in the both node
```
	wget http://www.mellanox.com/downloads/UFM/ufm_ha_6.1.1-2.tgz
```

2. Extract and install the UFM HA
```
tar -xvf ufm_ha_6.1.1-2.tgz
cd ufm_ha_6.1.1-2/
sudo ./install.sh -l /opt/ufm/files -d /dev/ubuntu-vg/ufm-dir-lv -p enterprise
```

3. Set password for user *hacluster* for both nodes
```
sudo passwd hacluster
```

4. Configure UFM HA
#### 1. On Standby (`hopper-ufm-01`):

Bash

```
sudo ufm_ha_cluster config -r standby \
  -l 192.168.12.32 \
  -e 192.168.12.14 \
  -p R0bust123456! \
  --enable-single-link \
  --no-vip
```

#### 2. On Master (`hopper-ufm-02`):

Bash

```
sudo ufm_ha_cluster config -r master \
  -l 192.168.12.14 \
  -e 192.168.12.32 \
  -p R0bust123456! \
  -i 192.168.12.20 \
  --enable-single-link
```

To check HA cluster status please use: ufm_ha_cluster status

Once DRBD sync is completed, you can start HA cluster using: ufm_ha_cluster start

UFM HA Status
<img width="700" height="666" alt="image" src="https://github.com/user-attachments/assets/e339cab5-1c02-47e7-b564-6d2d64946304" />


Replication DRBD status
```
robust@hopper-ufm-02:~$ drbdadm status
ha_data role:Primary
  disk:UpToDate
  peer role:Secondary
    replication:Established peer-disk:UpToDate

```



## Recommendation

Since the networks are totally isolated and you cannot ping the iDRAC from the server OS, **STONITH will fail** in its current state. For one node to "fence" (reboot) the other, there must be a communication path from the host OS to the other node's iDRAC IP.


---
### What happens if you skip STONITH?

If you cannot bridge the networks, you must leave `stonith-enabled=false`.

**The Risk:** If your network link between the servers blips but both servers stay powered on, they will both try to become "Master" (Split-Brain). They will both try to mount the same DRBD volume and write to the database. This usually results in **total data corruption** of your UFM configuration.

## Fail Over Test

Since UFM running a **Single Link** without **STONITH**, testing needs to be done with a bit of caution. Without a fencing mechanism (STONITH), the cluster can't "kill" a stuck node, so we want to avoid creating a "Split-Brain" scenario where both servers try to own the data at once.

---

### Preparation: The "Command Center"

Open three terminal windows to monitor the cluster in real-time:

1. **Terminal 1 (on Node 02):** `watch -n 1 sudo ufm_ha_cluster status`
    
2. **Terminal 2 (on your PC):** `ping 192.168.12.20` (to watch the Virtual IP move).
    
3. **Terminal 3 (on Node 01):** `sudo journalctl -f -u pacemaker` (to see the "brain" of the cluster thinking).
    

---

### Test 1: The Graceful "Soft" Failover (Safest)

This test checks if Pacemaker can correctly unmount the disk, move the IP, and start the service on the other node when commanded.

**The Command:**

On the current Master (`hopper-ufm-02`), run:

Bash

```
sudo pcs node standby 192.168.12.14
```

**What to look for:**

- **VIP:** ping should drop for a few seconds and then resume.
    
- **DRBD:** In the status, `hopper-ufm-02` should become `Secondary` and `hopper-ufm-01` should become `Primary`.
    
- **Service:** The `ufm-enterprise` resource should stop on Node 02 and start on Node 01.
    
- **Access:** Refresh your browser at `https://192.168.12.20`. You should be able to log in.
    

**To Revert:**

Bash

```
sudo pcs node unstandby 192.168.12.14
```

---

### Test 2: The Service Failure Test

This tests if the cluster detects when the UFM application itself crashes, even if the server is fine.

1. Find the UFM container ID: `sudo docker ps`.
    
2. **Kill the container:** `sudo docker kill ufm`.
    
3. **Watch:** Pacemaker should detect the service is down. Because it is a "Resource," it will first try to restart it on the same node. If it fails multiple times, it will move the entire group (Disk + IP + UFM) to Node 01.
    

---

### Test 3: The "Hard" Failover (Simulated Crash)

This is the most realistic test, but it is the riskiest without STONITH. This simulates a power failure or a kernel panic.

**The Action:**

Reboot the current Master node:

Bash

```
sudo reboot
```

**What happens without STONITH:**

- Node 01 will stop receiving "heartbeats" from Node 02.
    
- After the timeout (usually 15-30 seconds), Node 01 will say "I am alone, I will take over."
    
- Node 01 promotes DRBD to Primary, mounts the disk, and brings up the VIP.
    

**The Danger:** When Node 02 comes back up after the reboot, it must **not** try to start UFM. Since it was rebooted, the cluster services should ideally start in a "follower" mode. If both nodes try to be Primary when Node 02 returns, you will have a **Split-Brain**.

---

### Key Metrics to Monitor

|**Feature**|**Expected Behavior**|
|---|---|
|**Failover Time**|The VIP should be reachable again within 30–60 seconds.|
|**Data Integrity**|Any configuration change made on Node 02 before the test should be visible on Node 01.|
|**DRBD State**|Should stay `UpToDate/UpToDate`. If it says `Consistent` or `Inconsistent`, the sync is broken.|

---

### Why the "Single Link" is your biggest risk

In your current setup, the "heartbeat" (the cluster's pulse) and the "data replication" (DRBD) are sharing the same wire.

- If that one cable is pulled, both nodes will think the other is dead.
    
- Without STONITH, they will **both** try to be Master.
    
- This is why that **second link** you are working on is so important; it provides a "second opinion" so the cluster knows the difference between a cable failure and a server crash.




#### Screenshot

1- Before failover
<img width="776" height="953" alt="image" src="https://github.com/user-attachments/assets/9585a9c1-81d7-423b-86fd-fb923a6b6223" />


- Failover test 1 - sudo pcs node standby 192.168.12.14
<img width="778" height="938" alt="image" src="https://github.com/user-attachments/assets/ae775989-3b3c-4b0e-9dc1-f5013c0c1a33" />

<img width="1452" height="751" alt="image" src="https://github.com/user-attachments/assets/98c0bd3e-4221-4074-a185-a9d8e8f0bdc3" />

<img width="712" height="692" alt="image" src="https://github.com/user-attachments/assets/1adfa6a5-aca4-4d6a-9463-8d12a7b4a6b2" />



2. Failover test 2 - kill the container
<img width="707" height="687" alt="image" src="https://github.com/user-attachments/assets/da04965b-6be6-4526-b607-d3faca07c8bc" />


	- The ufm not starting to secondary node. This may risk to split brain since the resource agent is not configure yet with STONITH. 
	- The ufm ha not send the instruction to the "hang node" to reboot.
	- To rollback, need do some constraint clearance
	Verify the constraint
<img width="585" height="128" alt="image" src="https://github.com/user-attachments/assets/12131066-fb93-41d5-8620-60615cf69e5c" />


	 Clear constraint
<img width="617" height="307" alt="image" src="https://github.com/user-attachments/assets/f4bff8ce-e890-4446-97d8-00a9f53ce9aa" />

	 
		
3. Failover test 3 - reboot current master
<img width="701" height="718" alt="image" src="https://github.com/user-attachments/assets/3ab56afc-c01d-45ba-85ec-7988a5e8bc90" />


After the node back to online
<img width="695" height="724" alt="image" src="https://github.com/user-attachments/assets/833bf67d-1ee4-438b-8784-6c8081c42643" />



Rollback

<img width="706" height="882" alt="image" src="https://github.com/user-attachments/assets/376606cf-a05b-4dfa-9c86-bf8c84517006" />


