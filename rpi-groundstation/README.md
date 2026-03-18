1. apple hotspot doesnt support mDNS (so can't use .local address). It also isolates devices on the hotspot network, so they can't talk to each other. This makes it impossible to connect to the RPi via its local IP address when using the apple hotspot.
   - solution: Use RPI Flasher to configure a network for the RPi that it will connect to on first boot. After flashing, insert the SD card to a computer and paste the following to install tailscale on the RPI at the end of the ``user-data`` file, at the end of the file in the bootfs partition. Make sure to change the hostname and authkey to your own values. The authkey can be generated from the Tailscale admin console. With this configuration, on first boot, the RPI will connect to the configured wifi network, install tailscale, and connect to the tailscale network with the specified hostname and authkey: 
```yaml
runcmd:
  - [ sh, -c, curl -fsSL https://tailscale.com/install.sh | sh ]
  - [ sh, -c, sudo hostnamectl hostname YOUR_HOSTNAME_HERE ]
  - [ tailscale, up, --ssh, --hostname=YOUR_HOSTNAME_HERE, --authkey=tskey-auth-KEY_HERE ]
```
 This allows for bypassing the apple hotspot's limitations (or other possible network limitations) and still being able to connect to the RPI remotely. **Even if you don't use apple hostpot** (i.e. you use an android/windows device's hotspot) this is still necessary to allow for remote access to the RPI without needing to know its local IP address or needing to connect from a different network.

2. SSH into it and change the root password. This is required to use Ansible's become method with sudo.

3. Run the bootstrap playbook to set up tailscale and clone the git repository with the rest of the ansible code. This will allow for remote access to the RPI via tailscale and also set up the necessary files for running the rest of the ansible playbooks.
    - first run: ansible-playbook -i inventory.yml bootstrap.yml --ask-pass --ask-become-pass
