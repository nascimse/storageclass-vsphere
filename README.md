# storageclass-vsphere
# Como configurar StorageClass no Kubernetes para provisionar no vSpere
# Procedimentos para configurar StorageClass e PersistentVolumClaim no vSphere
# Referencias: https://vmware.github.io/vsphere-storage-for-kubernetes/documentation/existing.html

01 - Definir o arquivo de configuração vsphere.conf, este arquivo deve ficar em uma área que seja montada no kubernetes
     neste exemplo o arquivo está em: /etc/kubernetes/pki/cloud-config/

    [Global]

    [VirtualCenter "host.net"]
    user = "administrator@vsphere.local"
    password = "XYZ@!xyz"
    port = "443"
    thumbprint = "G5:2T:G4:4E:FE:32:D1:24:4F:25:39:GR:63:19:2B:5D:29:D1:R1:2C"
    datacenters = "dc-vcenter"  

    [Workspace]
    server = "host.net"
    datacenter = "dc-vcenter"
    default-datastore = "ds-vcenter"
    resourcepool-path = "rs-vcenter/rp-app"
    folder = "kubevol"

    [Disk]
    scsicontrollertype = pvscsi

    #Informações detalhadas dos parâmetros
    [Global] # Qualquer coisa na seção Global será aplicada a todos os vCenters no arquivo de configuração, a menos que seja substituído na seção VirtualCenter

    [VirtualCenter "host.net"] # O endereço IP do seu servidor vCenter
    user = "administrator@vsphere.local" # O usuário do vCenter para se autenticar
    password = "XYZ@!xyz" # A senha do usuário do vCenter acima
    port = "443" # A porta da API do vCenter - geralmente 443
    insecure-flag = "1" # Não verifique o certificado SSL no vCenter, NÃO ESTAMOS USANDO, DEVE SER USADO O PARÂMETRO (thumbprint), ISSO PELO FATO DO CERTIFICADO DO VCENTER SER AUTO ASSINADO
    datacenters = "dc-vcenter" # # O nome do seu datacenter no vCenter, onde os nós do K8s residem

    [Workspace] # Isso define onde o armazenamento do contêiner será provisionado (geralmente o mesmo que o vCenter acima) ao usar o SPBM
    server = "host.net" # O endereço IP do seu servidor vCenter para operações de provisionamento de armazenamento
    datacenter = "dc-vcenter" # O datacenter para provisionar VMs temporárias para provisionamento de volume
    default-datastore = "ds-vcenter" # O armazenamento de dados padrão para provisionar VMs temporárias para provisionamento de volume
    resourcepool-path = "rs-vcenter/rp-app" # O pool de recursos para provisionar VMs temporárias para provisionamento de volume
    folder = "kubevol" # A pasta da VM em que as VMs do Kubernetes estão, no vCenter

    [Disk] # Esta seção é obrigatória
    scsicontrollertype = pvscsi # Define o controlador SCSI em uso nas VMs - deixe como está

    OBS: Para conseguir o valor do parâmetro (thumbprint), siga os passos
         Referencia: https://vmware.github.io/vic-product/assets/files/html/1.3/vic_vsphere_admin/obtain_thumbprint.html

    01.01 - Você pode usar o SSH e o OpenSSL para obter a impressão digital do certificado para uma instância do vCenter Server Appliance ou um host ESXi.
        
        - Use o SSH para conectar-se ao host do vCenter Server Appliance ou ESXi como usuário root.
            $ ssh root@vcsa_or_esxi_host_address

        - Use openssl para visualizar a impressão digital do certificado.
        
            Appliance do vCenter Server:
                openssl x509 -in /etc/vmware-vpx/ssl/rui.crt -fingerprint -sha1 -noout
                Resultado: SHA1 Fingerprint=G5:2T:G4:4E:FE:32:D1:24:4F:25:39:GR:63:19:2B:5D:29:D1:R1:2C

            Host ESXi:
                openssl x509 -in /etc/vmware/ssl/rui.crt -fingerprint -sha1 -noout
                Resultado: Resultado: SHA1 Fingerprint=G5:2T:G4:4E:FE:32:D1:24:4F:25:39:GR:63:19:2B:5D:29:D1:R1:2C

        - Copie a impressão digital do certificado para uso na opção --thumbprint dos comandos vic-machine ou para defini-la como uma variável de ambiente.

02 - No master do Kubernetes
    02.01 - Inclua os seguintes linhas na configuração do serviço kubelet (geralmente no arquivo de configuração systemd), 
            bem como os arquivos de manifesto do contêiner controller-manager e api-server no nó master (geralmente em /etc/kubernetes/manifests).

        --cloud-provider=vsphere
        --cloud-config=/etc/kubernetes/vsphere.conf

        No arquivo /etc/kubernetes/manifests/kube-controller-manager.yaml procure o bloco abaixo para inserir
        as linhas no final deste bloco

            spec:
            containers:
            - command:
                - kube-controller-manager
                - --allocate-node-cidrs=true
                - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
                - --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf
                    .
                    .
                    .
                - --cloud-provider=vsphere     
                - --cloud-config=/etc/kubernetes/pki/cloud-config/vsphere.conf
    
    02.02 - No arquivo /etc/kubernetes/manifests/kube-apiserver.yaml procure o bloco abaixo para inserir
            as linhas no final deste bloco

            spec:
            containers:
            - command:
                - kube-apiserver
                - --advertise-address=10.35.40.121
                - --allow-privileged=true
                - --authorization-mode=Node,RBAC
                    .
                    .
                    .
                - --cloud-provider=vsphere
                - --cloud-config=/etc/kubernetes/pki/cloud-config/vsphere.conf
    
    02.03 - Modifique o kubelet service
        Não modificar o arquivo do systemd diretamente (o que provavelmente será sobrescrito nas atualizações). 
        Em vez disso, há um arquivo de ambiente (/var/lib/kubelet/kubeadm-flags.env), adicione as informações abaixo:

        --cloud-provider=vsphere
        --cloud-config=/etc/kubernetes/vsphere.conf

            KUBELET_KUBEADM_ARGS=--cgroup-driver=systemd --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1 --cloud-provider=vsphere --cloud-config=/etc/kubernetes/pki/cloud-config/vsphere.conf
    
    02.04 - Depois das alterações execute os comandos abaixo para efetivar:
        systemctl daemon-reload
        systemctl restart kubelet

        Verfique as logs se ocorreu algum problema
            kubectl logs -n kube-apiserver-node-k8s02 -n kube-system
            kubectl logs -n kube-controller-manager-node-k8s02 -n kube-syste
        
                
03 - Nos workers do Kubernetes
    03.01 - Modifique o kubelet service
        Não modificar o arquivo do systemd diretamente (o que provavelmente será sobrescrito nas atualizações). 
        Em vez disso, há um arquivo de ambiente (/var/lib/kubelet/kubeadm-flags.env), adicione as informações abaixo:

        --cloud-provider=vsphere

            KUBELET_KUBEADM_ARGS=--cgroup-driver=systemd --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1 --cloud-provider=vsphere
    
    03.02 - Depois da alteração execute os comandos abaixo para efetivar:
        systemctl daemon-reload
        systemctl restart kubelet

        Verfique as logs no master se ocorreu algum problema
            kubectl logs -n kube-apiserver-node-k8s02 -n kube-system
            kubectl logs -n kube-controller-manager-node-k8s02 -n kube-syste

04 - Atualizar os campos ProviderID no master e workers

    04.01 - Baixe e instale o client govc:
        Referencia: https://www.unixarena.com/2019/06/govmomi-installing-and-configuring-govc-cli-for-vsphere.html/

    04.02 - Como root exporte as variáveis:
        export GOVC_URL="https://host.net"
        export GOVC_USERNAME=administrator@vsphere.local
        export GOVC_PASSWORD=XYZ\@\!xyz  # OBS: Coloque \ nos caracteres especiais
        export GOVC_INSECURE=1
        export DATACENTER=dc-vcenter
        export FOLDER=kubevol

    04.03 - Identifique onde estão as VM's com o comando 'govc ls'. O objetivo é identificar o UUID da VM que será cadastrada
        posteriormente no kubernetes:

        [root@node-k8s02 ~]# govc ls /dc-vcenter/vm/VMs/Linux/k8s
        /dc-vcenter/vm/VMs/Linux/k8s/node-k8s05
        /dc-vcenter/vm/VMs/Linux/k8s/node-k8s02
        /dc-vcenter/vm/VMs/Linux/k8s/node-k8s03
        /dc-vcenter/vm/VMs/Linux/k8s/node-k8s04

        govc vm.info -json -dc=dc-vcenter -vm.ipath="/dc-vcenter/vm/VMs/Linux/k8s/node-k8s05" -e=true | jq -r ' .VirtualMachines[] | .Config.Uuid' | awk '{print toupper($0)}'
422B3487-9DF0-524C-ED9B-469083CBC871
   
        Faça o procedimento para encontrar o UUID de todas as VM's, depois execute o comando abaixo para configurar o UUID no kubernetes

        Como o UUID identificado foi da VM node-k8s05, vamos configurar ele no node node-k8s05 do kubernetes
        kubectl patch node node-k8s05 -p '{"spec":{"providerID":"vsphere://422B3487-9DF0-524C-ED9B-469083CBC871"}}'

        OBS: MUITA ATENÇÃO AO EXECUTAR O COMANDO 'kubectl patch node', SE FOR CONFIGURADO O UUID NO NODE ERRADO NÃO É 
        POSSIVEL FAZER A DELEÇÃO DO MESMO, SERÁ NECESSÁRIO FAZER OS PROCEDIMENTO:
            - BAKCUP DA CONFIGURAÇÃO DO NODE
            - CORDON DO NODE
            - DRAIN DO NODE
            - REMOVER O NODE
            - CRIAR O NODE NOVAMENTE COM O BACKUP FEITO E COM O UUID CORRETO
            - UNCORDON DO NODE

05 - Criação do StorageClass e PersistentVolumClaim

    05.01 - Criação do StorageClass

        Criar o yaml vsphere-storageclass.yaml

        cat <<EOF > vsphere-storageclass.yaml
        kind: StorageClass
        apiVersion: storage.k8s.io/v1
        metadata:
          name: vsphere-fast
          annotations:
        provisioner: kubernetes.io/vsphere-volume
        parameters:
            datastore: ds-vcenter
            diskformat: zeroedthick
            fstype: ext4
        reclaimPolicy: Delete
        EOF

        kubectl apply -f vsphere-storageclass.yaml

        kubectl get sc
        NAME                PROVISIONER                    AGE
        vsphere-fast        kubernetes.io/vsphere-volume   6d20h

    05.02 - Configuração do PersistentVolumClaim deve ser feito no Deployment ou StatefulSet conforme exemplo abaixo:

         containers:
                      .
                      .
                      .
              volumeMounts:
              - mountPath: "/usr/share/elasticsearch/data"
                name: data
        volumeClaimTemplates:
        - metadata:
            name: data
          spec:
            storageClassName: vsphere-fast
            accessModes: [ ReadWriteOnce ]
            resources:
              requests:
                storage: 20Gi
          
          kubectl get pvc -n elasticsearch
          NAME                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
          data-elasticsearch-0   Bound    pvc-1b840240-8021-11ea-8400-005056ab2137   20Gi       RWO            vsphere-fast       5d19h
          data-elasticsearch-1   Bound    pvc-2dd6cb46-8021-11ea-8400-005056ab2137   20Gi       RWO            vsphere-fast       5d19h
          data-elasticsearch-2   Bound    pvc-3eb9f112-8021-11ea-8400-005056ab2137   20Gi       RWO            vsphere-fast       5d19h
  