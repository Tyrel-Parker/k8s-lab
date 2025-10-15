# k8s-lab
Libvert vms established using vagrant for the testing and practice of kubernetes administration. 

# Make sure that libvert and vagrant are installed.
This assumes that KVM is installed and running.

# Vagrant
vagrant up              // might show some libvirt_ip_command errors due to an older fog-libvirt library being used. IGNORE.
vagrant status
virsh list --all        
virt-manager            // gui vm manager
vagrant ssh control     // will setup ssh connection to the control node
vagrant ssh worker1     // ssh connection to worker1
