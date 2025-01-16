A basic Proxmox & Rocky 9 test/learning environment for [Ansible Molecule](https://ansible.readthedocs.io/projects/molecule/). This was developed on a Rocky 9 host. A successful deployment of the test environment should result in you being able to run a basic molecule test. In other words - once everything is working, molecule will be able to spin up a virtual machine on the Proxmox host, connect to it via SSH, deploy the test role, and then perform a basic verification. Once the test has finished molecule will shutdown and destroy the VM.

**Prerequisites** :

- A Proxmox VE server
- Python >= 3.12
- Python venv module
- A Rocky host to run Ansible and Molecule on
- An SSH keypair

# Initial Setup

What follows are the steps you will need to follow to setup this test environment.

## Prepare Test Environment

**Create SSH Keypair**

- *If* an SSH keypair does not yet exist - create one: `ssh-keygen` + `Enter`, `Enter`, `Enter`.  This will create the keypair under `~/.ssh`

**Install required system packages**:

- On a Rocky machine where molecule is to run: `sudo dnf -y install git epel-release python3.12 python3.12-pip`

**Clone Repo and configure Python environment**:

- `git clone https://github.com/linuxuser332/molecule-proxmox-basic`
- `cd molecule-proxmox-basic`
- Create Python virtual environment: `python3.12 -m venv virtualenv`
- Switch to new Python environment: `source virtualenv/bin/activate`
- Ensure no errors, and that command prompt now shows `(virtualenv)`
- Ensure pip updated: `pip install --upgrade pip`
- Install molecule and the proxmox driver: `pip install molecule molecule-proxmox requests`

**Install Ansible modules**:

- `ansible-galaxy collection install community.general`

**Configure molecule.yml**:

The driver and platforms now need configuring. Using preferred text editor, set the following fields within `roles/makeTestDir/molecule/default/molecule.yml`: 

|Line|Field                |Description                               |
|----|---------------------|------------------------------------------|
|6   |`proxmox_secrets`      |Full path to `secrets.yml`              |
|7   |`node`                 |Hostname of Proxmox server              |
|9   |`ssh_identity_file`    |Full path to the private key            |
|16  |`template_name`        |Name of the template to deploy          |
|18  |`hostname`             |The name of the VM molecule will deploy |
|19  |`ssh_private_key_file` |Full path to the private key            |

## Proxmox Server Configuration
### Token Creation
Molecule (via a driver) will be connecting to Proxmox with a token, so it must be created in the web UI first:

- Logon to the Proxmox web UI
- Navigate to `Datacenter → API Tokens` and click the `Add` button
- Select the user which this token will be assigned to
- Enter a token ID.  We suggest `molecule`
- Untick `Privilege Separation`
- Click `Add`
- Note down the ID and the Secret value. They will be needed later

### Template VM Creation
A template VM must be created in Proxmox with the necessary configuration to allow it to join the network.  This test environment will be using Rocky 9, so a cloud image in qcow2 format is required. At the time of writing this can be found [here](https://rockylinux.org/download).

Once the image is downloaded, `scp` the file onto the Proxmox server, and into the directory `/var/lib/vz/template/qemu`. This should now give the file `/var/lib/vz/template/qemu/Rocky-9-GenericCloud-Base.latest.x86_64.qcow2`

**Create the template VM**:

- Logon to the Proxmox web UI
- In the tree on the left, right click the hostname of your proxmox server and select `Create VM`
- Enter the name `Rocky-9-Template` and click `Next`
- On the `OS` pane, select `Do not use any media` and click `Next`
- On the `System` pane, tick the `Qemu Agent` box and click `Next`
- When using this template, you may often need more space than the qcow2 disk provides. Therefore, the default disk can be left in place. Click `Next`
- On the `CPU` pane, increase the cores to 2 and click `Next`
- If you are not memory constrained, increase the assigned memory to 4096 and click `Next`
- Assuming you have a default bridge of `vmbr0`, leave the network settings unchanged and click `Next`
- Click `Finish`
- Note the ID of the newly created VM

**Assign the qcow2 image to the VM**:

- SSH onto the Proxmox server
- `cd /var/lib/vz/template/qemu`
- Using the ID of the new VM and assuming a storage pool named `local-lvm` exists, assign the image to the VM: `qm importdisk ID Rocky-9-GenericCloud-Base.latest.x86_64.qcow2  local-lvm`
- The image will now be uploaded into the storage and should display `successfully import disk....`

**Configure boot device**:

- In the Proxmox web UI - click on the new VM
- Click `Hardware` in the pane
- Click on the `Unused Disk 0` and then click the `Edit` button
- In the pop-up window, now click the `Add` button
- Now select `Options`  and from the list select `Boot Order`. Now  click `Edit`
- Using the grab handles on the left, drag the 10G disk so that it is option 1, and tick `Enabled`.   Click `OK`

**Configure Cloud-Init**:

- Select `Hardware` from the list
- Click the `Add` button and select `CloudInit Drive`
- Leave as IDE 0 and select the same storage as the VM is using (local-lvm)
- Select `Cloud-Init` from the list
- Using the UI, now set the following information:
-- User: `ansible`
-- SSH public key: Paste in the public key - typically `~/.ssh/id_rsa.pub` on the test host
-- A password is not strictly needed. If console login is needed however, set one in `Password`. Otherwise only key authentication will be available
-- Set `IP Config (net0)` select DHCP as VMs will need dynamic IP assignment
-- If `Upgrade packages` is selected the VM will run a full package update upon boot. This will slow down initial availability of the VM. As this is a test environment, deselect this.

**Convert to template**:
 
 - In the top right hand, click on `More` and select `Convert to template`

## Final Configuration
Now that the Proxmox token exists it can be configured in the file `vars/secrets.yml`:

- Edit the file vars/secrets.yml in the molecule directory structure (eg: `~/molecule-proxmox-basic/roles/makeTestDir/molecule/default/vars/secrets.yml)
`
- Populate the variables as required and save the file

# Molecule Run
If all is configured correctly, Molecule should now (via the molecule-proxmox driver) be able to create a test VM, apply the Ansible role `makeTestDir` and test that it runs successfully.  

- Change to the role directory eg: `cd ~/molecule-proxmox-basic/roles/makeTestDir`
- Issue the command `molecule create`.  This should result in Molecule creating the VM.
- If that was successful, you can now issue the other molecule commands. `molecule verify` will deploy the role and verify that it worked properly.
- `molecule idempotence` will ensure the role is idempotent. 
- When you are ready to destroy the VM run  `molecule destroy`.  This should result in the VM being deleted.
- An entire test run from start to finish can be run with `molecule test`


