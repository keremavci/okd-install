Herkese merhaba, bu yazıda VMware vSphere ortamına OKD 4.5 kurulumundan bahsedeceğim.

Kurulum senaryomuzda OKD clusterının internete çıkış izni olmadığı için air-gapped bir kurulum gerçekleştireceğiz.

> Yazıda kullanılan kaynaklara https://github.com/keremavci/okd-install adresinden erişebilirsiniz.

Redhat, CoreOS firmasını satın almasından sonra Openshift platformunda çok büyük değişiklikler oldu. OKD 4, işletim sistemi olarak Fedora CoreOS dağıtımını kullanıyor. Fedora CoreOS, çoğumuzun CoreOS Container Linuxdan bildiğimiz containerlar için minimal, otomatik güncellenen ve immutable bir işletim sistemi. OKD'nin tüm bileşenleri artık container ve operatorlar aracılığı ile sistemimize kuruluyor. Tüm bunlar birleşince OKD hızlı kurulan, hızlı upgrade edilen, stabil bir K8s dağıtımı ve hatta PaaS çözümü olarak karşımıza çıkıyor.

Kurulum senaryomuz https://www.equinix.com adresinden aldığımız bir bare metal sunucu üzerinde Vsphere 6.7 kurulu olan sunucuda gerçekleşecek. Ben burada aldığım sunucuya öncesinde vCenter kurulumunu gerçekleştirdim. vCenter kurulumu için https://computingforgeeks.com/install-vcenter-server-appliance-on-esxi-host/ adresindeki yönergeleri takip ettim.


Devamında ise işlerimi aynı networkden daha rahat halletmek için bir tane ubuntu jumpbox sunucusu ayağa kaldırdım. Bu adımda da ufak bir trick, vCenter isoları daha hızlı download etmek için https://metal.equinix.com/developers/docs/guides/vmware-esxi/ adresinde de tavsiye ettiği gibi vSphere ssh ile direk girip gerekli isoları wget ile çekebiliyoruz.

https://metal.equinix.com/developers/docs/guides/vmware-esxi/

![Alt text](docs/images/vcenter.jpg?raw=true "vCenter")



Ortamımızda OKD kurulumuna başlamadan önce bir Docker Image Registry, DHCP ve DNS Serverın kurulu olması gerekiyor. 


## DNS, DHCP ve Registry Kurulumu

Senaryomuzda hızlıca sonuca ulaşmak için DNS, DHCP ve Registry kurulumlarımız container içinde hizmet verecek. Bu containerları dhcp-dns-registry.yml dosyası ile ayağa kaldırıyorum..

>Registry için Sertifika oluşturmam gerekiyor. Bunun için aşağıdaki linteki dökümanı takip ederek kök sertifikalarımı daha sonrasında da registry için kullacağım sertifikları oluşturuyorum. 
https://gist.github.com/fntlnz/cf14feb5a46b2eda428e000157447309


![Alt text](docs/images/ubuntu-docker-ps.jpg?raw=true "Ubuntu docker ps")

Gerekli containerları ayağa kaldırdıktan sonra nslookup ile dns serverımdan gerekli kayıtları sorguluyorum. api.cluster_name.base_domain ve *.apps.cluster_name.base_domain kayıtları kurulum içinde bize gerekiyor.

![Alt text](docs/images/dns.jpg?raw=true "nslookup")


## Openshift Installer ve OC'nin İndirilmesi

openshift-installer ve oc'yi  https://github.com/openshift/okd/releases sayfasından indeirebilirsiniz. Diğer bir alternatifiniz de https://origin-release.apps.ci.l2s4.p1.openshiftapps.com/ adresinde oc toolu ile dilediğiniz versiyonu indirmek.

>Ben yazıya başlarken 4.6 versiyonunu kurmak istiyordum fakat 4.6 versiyonunda container imajlarının mirror sha değerlerinde tutarsızlık oluşuyor. Şu an hala açık bir issue mevcut. https://github.com/openshift/okd/issues/402 Bu nedenle okd:4.5.0-0.okd-2020-10-15-235428 versiyonunu kuracağım.


oc adm release extract --tools quay.io/openshift/okd:4.5.0-0.okd-2020-10-15-235428


## vCenter Sertifikalarının Sunucuya Eklenmesi
Kuruluma başlamadan önce vCenter Root sertifikalarının kurulumu başlatacağımız sunucuya eklenmesi gerekiyor. Ben bunun için aşağıdaki komutları kullanıyorum.

```
export VCENTER_URL=https://145.40.64.187
curl -vkLO $VCENTER_URL/certs/download.zip
unzip download.zip
cd certs/lin
for i in `ls;do mv $i $i.crt;done
cd -
sudo cp -R lin/ /usr/local/share/ca-certificates/
sudo chmod 0644 -R /usr/local/share/ca-certificates/lin/
sudo update-ca-certificates
```

## SSH Key'lerin Oluşturulması

```
ssh-keygen -t rsa -b 4096 -c "avci.kerem@gmail.com"
```
komutu ile kurulumdan sonra oluşturulacak vmlere eklenecek public key'i ve bu vmlere ssh ile bağlanırken kullanacağımız private key dosyalarını oluşturuyoruz.


## Imajların Mirrorlanması
İmajların mirrolanması yine oc aracılığı ile yapılıyor. Aşağıdaki komutlarla
```
OKD_RELEASE=4.5.0-0.okd-2020-10-15-235428
LOCAL_REGISTRY=registry.okd.keremavci.dev/openshift
LOCAL_REPO=okd
PRODUCT_REPO=openshift
RELEASE_NAME=okd
oc adm release mirror \ 
--from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OKD_RELEASE} \ 
--to=${LOCAL_REGISTRY}/${LOCAL_REPO} \
--to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPO}:${OKD_RELEASE}
```


Bu aşama sonunda bize bir çıktı verecek. Bu çıktı önemli. Bu çıktıyı biraz sonra oluşturacağımız install-config.yaml dosyasında kullanacağız.

```
imageContentSources:
- mirrors:
  - registry.okd.keremavci.dev/openshift/okd
  source: quay.io/openshift/okd
- mirrors:
  - registry.okd.keremavci.dev/openshift/okd
  source: quay.io/openshift/okd-content

```


## Install-Config Dosyasının Oluşturulması

```
openshift-install create install-config --dir okd-install-config
```
komutu ile install-config.yaml dosyasını interaktif bir şekilde oluşturuabilirsiniz. İlerleyen süreçte başka clusterlara ihtiyacınız oluduğunda aynı dosyayı değiştirerek kullanabilirsiniz.
Ben okd-install-config içerisinde oluşturduğum install-config.yaml dosyasına biraz önce oc adm release mirror komutu ile oluşan çıktıyı ve additionalTrustBundle altınada registry sertifikalarında kullandığım root sertifikasınıda ekliyorum.Oluşan install-config.yaml dosyasının içeriğine bu dökümanın git reposunda ulaşabilirsiniz.


![Alt text](docs/images/okd-install-config.jpg?raw=true "create-config")

>Burada dikkat edilmesi gereken, Pull Secret alanına local registryninzde authentication kullanmasanızda buraya, 
`{"auths":{"fake":{"auth": "bar"}}}` gibi sahte bir değer yazmanız gerekiyor.

>Sizin kurulumunda ek olarak proxy gibi yada oluşturulacak vmlerin sayısı gibi özelleştirmeye ihtiyaç var ise https://docs.okd.io/latest/installing/installing_vsphere/installing-vsphere-installer-provisioned-network-customizations.html linkinden yararlanabilirsiniz.



## Kurulumun Başlatılması

Biraz önce son düzenlemeleri yaptığım install-config.yaml dosyasını bu senaryoda okd-cluster klasörüne kopyalayarak,
```openshift-install create cluster -dir okd-cluster```
komutu ile kurulumu başlatıyorum.


![Alt text](docs/images/okd-install-command.jpg?raw=true "create-cluster")


>Kurulum ilk önce Fedora CoreOS imajını indirip VCenter'a ekleyerek ile başlıyor. Devamında bu senaryoda 3 master ve 1 bootstrap node ayağa kaldırıyor. Bu bootstrap sunucusu masterlar için ignition configleri serve ediyor ve gerekli diğer kurulumları gerçekleştiriyor.
Kurulum sırasında bir sorun olup olmadığını izlemek için, install-config oluşturuken yaratılan private key ile bootstrap sunucusuna bağlanıp journalctl ile kurulumunu kontrol edebilirsiniz.

## Son Kontroller ve Dashboard
```openshift-install create cluster -dir okd-cluster``` komutu ile
kurulum tamamlanınca bize console adresini, kubeadmin şifresi gibi bilgileri veriyor.

Kurulumdan sonrasında ,
```
oc get clusteroperators
oc get clusterinfo
oc get nodes
```
komutları ile clusterınızın durumunu görüp, bu senaryoda okd-cluster/auth altında bulunan kubeadmin-password dosyasında yazan kubeadmin ile dashboarda giriş yapabilirsiniz..

![Alt text](docs/images/oc-cluster-info.jpg?raw=true "oc-cluster-info")

![Alt text](docs/images/okd-dashboard.jpg?raw=true "okd-dashboard")


