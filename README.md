<br />
<div align="center">
  <h3 align="center">Ansible reporting with ARA: Ansible Run Analysis</h3>
  <p align="center">
    Detailed analysis of the Ansible playbook runs
</div>

# ansible_optimize
### Installing Ansible
To begin exploring Ansible as a means of managing our various servers, we need to install the Ansible software on at least one machine.
To get Ansible for CentOS 7, first ensure that the CentOS 7 EPEL repository is installed:
 ```sh
  sudo yum -y install epel-release
  ```
 Once the repository is installed, install Ansible with yum:
 ```sh
  sudo yum -y install ansible
  ```
 
 ### Optimizing Ansible
 #### Facts Gathering
 By default, when Ansible connects to a host, it collects information such as system information (processors, OS, CPUs), network connectivity, devices information etc.</br>
 This is known as "**facts**". This operation can be time-consuming, and should be avoided if not necessary, or optimized with a facts cache when necessary.<br/>
 I would suggest to let gathering enabled by default in your ansible.cfg file and disable it when you don’t need them directly in your playbooks :
  ```
  - hosts: all
  gather_facts: no
  ```
  
  Facts gathering can be improved using fact caching. <br/>
  It can use redis or create JSON files on your Ansible host. Following options in the ansible.cfg file will use smart gathering and facts caching with a local json file :
   ```
    gathering = smart
    fact_caching = jsonfile
    fact_caching_connection = /tmp
   ```
   
  The "**Smart**" option means each new host that has no facts discovered will be scanned, but if the same host is addressed in multiple plays it will not be contacted again in the playbook run. We will keep cache in JSON format in the "**tmp**" directory.
  
   #### ssh
   There are several SSH settings that can be tuned for a better performance. First, we should configure ControlPersist so connections to servers can be recycled and set PreferredAuthentications to “publickey” to not run into delays in servers that have GSSAPIAuthentication enabled.<br/>
   The most important setting for SSH is, in my opinion, “pipelining”, as it will reduce the number of SSH connections required to run some modules.<br/>
   Here is an example of ansible.cfg with such SSH optimizations :
   ```
    [ssh_connection]
    pipelining=True
    ssh_args = -C -o ControlMaster=auto -o ControlPersist=3600s -o PreferredAuthentications=publickey -o "PasswordAuthentication=no" -o "StrictHostKeyChecking=no" -o "UserKnownHostsFile=/tmp/ssh_cach" -o "ConnectTimeout=60"
   ```
   
   #### Tuning parameters 
   Here the official documentation about [tuning performance in Ansible](https://www.ansible.com/blog/ansible-performance-tuning).<br/>
   **forks** : how many maximum parallel connections can be initiated for manage nodes from control node to execute ansible commands, 
   default fork value is 5.<br/>
   **Usage** : When you need to manage how many hosts may get affected on same time.<br/>
   **Example** : We have 8 hosts (server_1, server_2, server_3, server_4, server_5, server_6, server_7, server_8) in inventory & 2 tasks in playbook (assuming default forks = 5)<br/>
   
 #### forks = 5
	First task process on 5 hosts (server_1, server_2, server_3, server_4, server_5) simultaneously, after that 5 nodes completed task process of rest of hosts (server_6, server_7, server_8). Then Second task process on 5 hosts (server_1, server_2, server_3, server_4, server_5) simultaneously, after that 5 hosts completed task process of rest of hosts (server_6, server_7, server_8). This is how the cycle continues in 5 batches of hosts until the tasks are completed, let's assume processing time for each task is 10 seconds;
    Time taken for 1st task = 10s (server_1, server_2, server_3, server_4, server_5) + 10s (server_6, server_7, server_8)
    Time taken for 2nd task = 10s (server_1, server_2, server_3, server_4, server_5) + 10s (server_6, server_7, server_8)
    Total time taken for playbook = 40s
	
#### forks = 10
	First task process on 8 hosts (server_1, server_2, server_3, server_4, server_5, server_6, server_7, server_8) simultaneously, let's assume processing time for each task is 10 seconds;
	Time taken for 1st task = 10s (server_1, server_2, server_3, server_4, server_5, server_6, server_7, server_8)
	Time taken for 2st task = 10s (server_1, server_2, server_3, server_4, server_5, server_6, server_7, server_8)
	Total time taken for playbook = 20s
	
	NB : You should care because this may affect the performance of the Ansible controller if the tasks are executed against a large number of hosts.
  
   
   
