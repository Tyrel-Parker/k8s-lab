# k8s-lab
Libvert vms established using vagrant for the testing and practice of kubernetes administration. 

# Make sure that libvert and vagrant are installed.
This assumes that KVM is installed and running.

1. Install KVM and libvirt
- sudo apt update  
- sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-manager

2. Install additional tools needed for vagrant-libvirt
- sudo apt install -y ruby-dev libvirt-dev build-essential libxml2-dev libxslt1-dev zlib1g-dev

3. Add your user to required groups
- sudo usermod -aG libvirt $USER  
- sudo usermod -aG kvm $USER

4. Start and enable libvirt service
- sudo systemctl start libvirtd  
- sudo systemctl enable libvirtd

5. Verify libvirt is running
- sudo systemctl status libvirtd

6. Finally
- **reboot**  
- **after reboot**  
- groups  
- **verify libvirt and kvm are in list**  
- kvm-ok  
- virsh list --all  
- **should be empty**  


# Vagrant
vagrant up              **might show some libvirt_ip_command errors due to an older fog-libvirt library being used. IGNORE.**  
vagrant status  
virsh list --all  
virt-manager            **gui vm manager**  
vagrant ssh control     **will setup ssh connection to the control node**  
vagrant ssh worker1     **ssh connection to worker1**  
