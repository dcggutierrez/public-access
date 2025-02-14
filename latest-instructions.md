-- nano /etc/network/interfaces

# allow-hotplug enp0s3
auto enp0s3
iface enp0s3 inet dhcp
auto enp0s3:0
iface enp0s3:0 inet static
	address 192.168.157.31
	netmask 255.255.252.0

-- nano /etc/hosts

192.168.157.31   dev-ph01.ustapps.com dev-ph01
192.168.157.32   dev-ph02.ustapps.com dev-ph02
192.168.157.33   dev-ph03.ustapps.com dev-ph03
192.168.157.34   dev-ph04.ustapps.com dev-ph04
192.168.157.35   dev-ph05.ustapps.com dev-ph05
192.168.157.36   dev-ph06.ustapps.com dev-ph06

192.168.157.1   dev-cluster-ph01.ustapps.com dev-cluster-ph01
192.168.157.2   dev-ldap-ph01.ustapps.com dev-ldap-ph01




  kubeadm join dev-cluster-ph01.ustapps.com:7443 --token pqblw2.qow5oax7up292pgi \
        --discovery-token-ca-cert-hash sha256:fd8145ac31535f6a1b3db378eb4e19e8bc227d831838d9e0fb4d76138343a8cd \
        --control-plane --certificate-key 7c9e1d6fc68a8afc58c2d790807438434f70cdfc338ad777ccd7c7105499aacb


kubeadm join dev-cluster-ph01.ustapps.com:7443 --token pqblw2.qow5oax7up292pgi \
  --discovery-token-ca-cert-hash sha256:fd8145ac31535f6a1b3db378eb4e19e8bc227d831838d9e0fb4d76138343a8cd \
  --control-plane --certificate-key 7c9e1d6fc68a8afc58c2d790807438434f70cdfc338ad777ccd7c7105499aacb \
  --cri-socket=unix:///var/run/cri-dockerd.sock --apiserver-advertise-address=192.168.157.32 -v=5