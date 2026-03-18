1. Apple hotspot still does not support mDNS and still isolates devices on the hotspot network, so local addressing remains unreliable for the Pi in that scenario.
   - first-boot solution: configure a normal Wi-Fi network for first boot and, when initial SSH reachability is uncertain, use Raspberry Pi cloud-init / `user-data` to install and join Tailscale before Ansible is involved
   - ongoing solution: after the Pi is reachable, let the Ansible bootstrap playbook install and manage Tailscale using the values in `vars/secrets.yml`
   - required variables now include:
     - `tailscale_authkey`
     - `tailscale_hostname`
     - optionally `tailscale_ssh`
     - `tailscale_lab_host` for the lab host name the Pi should forward to
     - `groundstation_fake_radio_enabled` if you want a test deployment to use the built-in fake telemetry fixture
   - this still allows bypassing hotspot limitations and gives the Pi a stable remote path to the lab host

2. SSH into it and change the root password. This is required to use Ansible's become method with sudo.

3. Run the bootstrap playbook after first-boot reachability is established. It will install/join Tailscale, clone the ground-station repo, write the Pi `.env`, and install the Pi forwarder service.
    - first run: ansible-playbook -i inventory.yml bootstrap.yml --ask-pass --ask-become-pass

4. For a same-day end-to-end test without radio hardware, set:
   - `groundstation_fake_radio_enabled: true`
   - optionally adjust `groundstation_fake_radio_repeat` and `groundstation_fake_radio_interval_s`
   - by default the fake source will read `/home/pi/leos-S26-ground-station/testdata/fake_radio_input.jsonl`
