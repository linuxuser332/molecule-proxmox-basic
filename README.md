A basic Proxmox & Rocky 9 test/learning environment for Ansible Molecule. This was developed on a Rocky 9 box, and a successful deployment of the test environment should result in you being able to run a basic molecule test. In other words, once everything is working molecule will be able to spin up a virtual machine on the Proxmox host, connect to it via SSH, deploy the test role, and then perform a basic verification. Once the test has finished, molecule will shutdown and destroy the VM.

Prerequisites (a guide to setting these up can be found below) :

- A Proxmox VE server
- Python >= 3.12 (that's what this role was developed with)
- Python venv module

# Initial Setup

What follows are the steps you will need to follow to setup this test environment.

## Proxmox Server Configuration
### Token Creation
Molecule will be connecting to Proxmox with a token, so you will need to create an access token for it to use.

- Logon to the Proxmox web UI
- Navigate to `Datacenter → API Tokens` and click the `Add` button
