[![Static Badge](https://img.shields.io/badge/1-HOME-red?style=for-the-badge)](./1%20-%20ROSA%20AWS.md)
[![Static Badge](https://img.shields.io/badge/2-RESUMO_ROSA-red?style=for-the-badge)](./2%20-%20Resumo%20ROSA.md)
[![Static Badge](https://img.shields.io/badge/3-Pré_Instalação-red?style=for-the-badge)](./3%20-%20Pre-Instalação%20-%20ROSA.md)
[![Static Badge](https://img.shields.io/badge/4-Criação_Cluster-red?style=for-the-badge)](./4%20-%20Criação%20Cluster.md)
[![Static Badge](https://img.shields.io/badge/5-Conta_Inicial-red?style=for-the-badge)](./5%20-%20Configurar%20Conta%20Inicial%20ROSA.md)
[![Static Badge](https://img.shields.io/badge/6-Permissões-red?style=for-the-badge)](./6%20-%20Configurar%20Permissões.md)
<!-- [![Static Badge](https://img.shields.io/badge/7-Acesso_com_GITHUB-red?style=for-the-badge)](./7%20-%20Configurar%20GitHub%20ROSA.md) -->

---
## Configurar classes de armazenamento adicionais

Conecte aplicativos aos tipos de volume do EBS que correspondam aos seus requisitos de custo e desempenho.

Os desenvolvedores solicitam armazenamento para seus aplicativos criando recursos de reivindicação de volume persistente (PVC) do Kubernetes. Os principais parâmetros de um PVC são o tamanho e o modo de acesso.

Como desenvolvedor, você não conhece os detalhes exatos da infraestrutura de armazenamento. Os administradores de cluster gerenciam essa infraestrutura, que pode ser composta por diversas tecnologias, como diferentes tipos de volumes Amazon Elastic Block Store (EBS). Os administradores expõem essas diferentes soluções de armazenamento por meio de classes de armazenamento para que os desenvolvedores possam selecionar a classe de armazenamento adequada em seus PVCs.

## Classes de armazenamento em um cluster ROSA
O processo de criação de cluster Red Hat OpenShift on AWS (ROSA) cria automaticamente classes de armazenamento para alguns tipos de volume EBS. Você pode listar as classes de armazenamento disponíveis em seu cluster executando os comandos `oc get StorageClass` ou `oc get sc`.

```
$ oc get sc
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE    ...
gp2             kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer ...
gp2-csi         ebs.csi.aws.com         Delete          WaitForFirstConsumer ...
gp3 (default)   ebs.csi.aws.com         Delete          WaitForFirstConsumer ...
gp3-csi         ebs.csi.aws.com         Delete          WaitForFirstConsumer ...
```

As seguintes classes de armazenamento estão disponíveis em um cluster ROSA:

* **gp2 e gp2-csi**
Essas classes de armazenamento provisionam volumes do tipo de volume EBS gp2 General Purpose versão 2. O tipo gp2 EBS usa volumes SSD.

* **gp3 e gp3-csi**
Essas classes de armazenamento provisionam volumes da última geração gp3 do tipo de volume EBS de uso geral. O tipo gp3 EBS usa volumes SSD. Ele fornece melhor rendimento que os volumes gp2 e é mais barato. OpenShift usa a classe de armazenamento gp3 por padrão, quando o recurso PVC do desenvolvedor não especifica explicitamente uma classe de armazenamento.

>[!NOTE]
>ROSA expõe os tipos de volume gp2 EBS como duas classes de armazenamento: `gp2` e `gp2-csi`.
>
>As classes de armazenamento *-csi usam plug-ins Container Storage Interface (CSI) para provisionar armazenamento. Com o padrão aberto CSI, os fornecedores de armazenamento podem fornecer plug-ins para suas soluções de armazenamento.
>
>Em versões futuras do ROSA, as classes de armazenamento que usam o plug-in CSI substituirão as classes legadas não CSI. Para os volumes `gp3` EBS, a classe de armazenamento gp3 já usa o plug-in CSI. Assim, as classes `gp3` e `gp3-csi` são iguais. OpenShift define ambas as classes para compatibilidade com versões anteriores.

### Criando classes de armazenamento
Algumas cargas de trabalho exigem armazenamento mais especializado do EBS do que os tipos de uso geral. Por exemplo, cargas de trabalho com uso intensivo de E/S, como sistemas de banco de dados, podem se beneficiar do tipo de volume EBS `io2`. Cargas de trabalho em lote que computam grandes conjuntos de dados frios podem usar o tipo de volume `sc1` EBS mais barato, que usa unidades de disco rígido (HDDs) mais lentas.>[!NOTE]
>Alguns tipos de volume do EBS podem não estar disponíveis na sua região da AWS. Revise a documentação da Amazon para verificar se o tipo que você planeja usar está disponível em sua região.
>
>Ou tente criar um volume no modo seco para verificar se você pode usar o tipo de volume em sua zona de disponibilidade.
>
>O comando a seguir mostra que o tipo de volume EBS io2 está disponível na zona de disponibilidade us-east-1a.
>
>```
>$ aws ec2 create-volume --dry-run --volume-type io2 --size 20 --iops 1000 --availability-zone us-east-1a
>
>Ocorreu um erro (DryRunOperation) ao chamar a operação CreateVolume: A solicitação teria sido bem-sucedida, mas o sinalizador DryRun está definido.
>```
>O comando a seguir mostra que o tipo de volume EBS io2 não está disponível na zona de disponibilidade sa-east-1a.
>
>
>```
>$ aws ec2 create-volume --dry-run --volume-type io2 --size 20 --iops 1000 --availability-zone sa-east-1a
>
>Ocorreu um erro (UnknownVolumeType) ao chamar a operação CreateVolume: tipo de volume não suportado 'io2' para criação de volume.
>```

Para usar os tipos de volume do EBS com suas cargas de trabalho, você deve expô-los como classes de armazenamento em seu cluster. O plug-in EBS CSI oferece suporte a todos os tipos de volume do EBS. O arquivo a seguir declara um recurso de classe de armazenamento que os desenvolvedores podem usar para provisionar volumes `io2` EBS:

```
apiVersão: storage.k8s.io/v1
tipo: StorageClass
metadados:
   nome: io2-csi
provisionador: ebs.csi.aws.com [ 1 ]
volumeBindingMode: WaitForFirstConsumer [ 2 ]
permitirVolumeExpansion: verdadeiro
parâmetros:
   csi.storage.k8s.io/fstype: ext4 [ 3 ]
   tipo: io2 [ 4 ]
   iops: "32000" [ 5 ]
   criptografado: "verdadeiro"  [ 6 ]
```

* 1 - Nome do plug-in CSI. Para volumes EBS, sempre defina o nome como ebs.csi.aws.com, que é o plug-in CSI fornecido pela Amazon e compatível com a Red Hat.

* 2 - O OpenShift provisiona o volume com o EBS somente depois de criar o pod que declara o PVC. Com essa configuração, o OpenShift cria o volume EBS na mesma zona de disponibilidade do pod.

* 3 - Sistema de arquivos que o plug-in CSI usa para formatar o volume.

* 4 - Nome do tipo de volume do EBS.

* 5 - O número máximo de operações de E/S por segundo (IOPS) que o volume deve fornecer. Essa opção é válida apenas para os tipos de volume EBS io1, io2 e gp3.

* 6 - O EBS criptografa os dados no volume.

Você deve configurar o parâmetro `volumeBindingMode` como `WaitForFirstConsumer`. Um volume EBS está vinculado a uma zona de disponibilidade e só pode ser acessado por nós nessa zona. Se o OpenShift provisionasse o volume antes de criar o pod, ele não saberia em qual zona de disponibilidade o pod será executado.

Ao criar um cluster ROSA, você pode configurá-lo para abranger diversas zonas de disponibilidade. A Red Hat recomenda esta configuração para clusters de produção. No entanto, os volumes do EBS estão vinculados a uma única zona de disponibilidade e a Amazon não os replica para outras zonas de disponibilidade. No caso de falha de toda uma zona de disponibilidade, o OpenShift não poderá mover os pods que usam volumes EBS da zona com falha.

O parâmetro iops é válido apenas para os tipos de volume EBS `io1`, `io2` e `gp3`. Em vez do parâmetro iops, você pode usar o parâmetro iopsPerGB, que especifica o número máximo de operações de E/S por segundo por GiB.

Para volumes EBS, o tamanho do armazenamento e o IOPS são dimensionados juntos. Em outras palavras, volumes maiores obtêm melhores desempenhos de IOPS.

Por exemplo, a Amazon define o desempenho máximo para o tipo de volume `io1` como 50 IOPS por GiB. Um volume io1 de 4 GiB tem um limite de 200 IOPS (50 x 4), enquanto um volume de 1 TiB tem um limite de 51.200 IOPS (50 x 1.024). Em alguns casos, para atingir o desempenho de IOPS necessário, talvez seja necessário atribuir mais espaço do que o necessário.

A criação de classes de armazenamento requer permissões de administrador de cluster. As permissões de administrador dedicadas não são suficientes.

>[!NOTE]
>Para otimizar o desempenho do volume, a Amazon recomenda usar um tipo de instância Amazon Elastic Compute Cloud (Amazon EC2) otimizado para EBS. Os tipos de instância do EC2 otimizados para armazenamento, como a família de instâncias i3, melhoram o desempenho de E/S e de taxa de transferência de leitura sequencial.
>
>A criação de um pool de máquinas para selecionar um tipo de instância EC2 para um grupo de nós de computação será abordada em uma seção posterior.

Provisionando volumes usando declarações de volume persistente
O exemplo a seguir declara um PVC que usa a classe de armazenamento anterior para solicitar um volume EBS io2 de 128 GiB:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dbdata
spec:
  storageClassName: io2-csi
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 128Gi
```

Você pode usar o AWS Management Console ou o comando aws para inspecionar o volume EBS resultante que o OpenShift cria a partir desta declaração. No volume no EBS, a tag `kubernetes.io/created-for/pvc/name` é definida como o nome do PVC. Adicione a opção `--filter` ao comando aws para filtrar os volumes EBS por esta tag.

O comando a seguir exibe os parâmetros do volume EBS que o OpenShift cria a partir do PVC `dbdata`:

```
$ aws ec2 describe-volumes
  --filters "Name=tag:kubernetes.io/created-for/pvc/name,Values=dbdata"
{
    "Volumes": [
        {
...output omitted...
            "Size": 128,
            "SnapshotId": "",
            "State": "in-use",
            "VolumeId": "vol-0b0102de7640bb98a",
            "Iops": 32000,
            "Tags": [
                {
                    "Key": "CSIVolumeName",
                    "Value": "pvc-fd7869ad-8d12-4683-8bd4-87217b306e59"
                },
                {
                    "Key": "kubernetes.io/created-for/pvc/name",
                    "Value": "dbdata"
                },
...output omitted...
            ],
            "VolumeType": "io2",
            "MultiAttachEnabled": false
        }
    ]
}
```

Armazenamento para nós OpenShift
Os volumes EBS são usados não apenas como recursos de volume persistente para OpenShift, mas também como discos para os nós do OpenShift. Todos os nós em um cluster ROSA usam os tipos de volume EBS gp3 para seus discos. O ROSA não permite escolher um tipo de volume diferente, mesmo para nós de cálculo criados por meio de pools de máquinas ROSA.

Em contraste com os clusters ROSA, você pode selecionar os tipos de volume para seus nós ao implantar um cluster OpenShift autogerenciado na AWS. Nesses clusters, você especifica o tipo de volume EBS no arquivo de configuração de instalação `install-config.yaml`.

Para grandes clusters OpenShift autogerenciados, uma necessidade comum é usar o tipo de volume io2 EBS para o armazenamento etcd com uso intensivo de E/S que é executado nos nós do plano de controle.

>## Referência
>
> - [Informações sobre volumes EBS em clusters ROSA](https://access.redhat.com/documentation/en-us/red_hat_openshift_service_on_aws/4/html-single/storage/index#configuring-persistent-storage)
> - [Informações sobre a criação de classes de armazenamento](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.12/html-single/post-installation_configuration/index#post-install-storage-configuration)
> - [Informações sobre a instalação de um cluster OpenShift autogerenciado na AWS](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.12/html-single/installing/index#installing-aws-customizations)
> - [Amazon EBS Volume Types](https://aws.amazon.com/ebs/volume-types/)
> - [Amazon EC2 Instance Types](https://aws.amazon.com/ec2/instance-types/)
> - [Amazon Elastic Block Store (EBS) CSI driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)

---
[![Static Badge](https://img.shields.io/badge/1-HOME-red?style=for-the-badge)](./1%20-%20ROSA%20AWS.md)
[![Static Badge](https://img.shields.io/badge/2-RESUMO_ROSA-red?style=for-the-badge)](./2%20-%20Resumo%20ROSA.md)
[![Static Badge](https://img.shields.io/badge/3-Pré_Instalação-red?style=for-the-badge)](./3%20-%20Pre-Instalação%20-%20ROSA.md)
[![Static Badge](https://img.shields.io/badge/4-Criação_Cluster-red?style=for-the-badge)](./4%20-%20Criação%20Cluster.md)
[![Static Badge](https://img.shields.io/badge/5-Conta_Inicial-red?style=for-the-badge)](./5%20-%20Configurar%20Conta%20Inicial%20ROSA.md)
[![Static Badge](https://img.shields.io/badge/6-Permissões-red?style=for-the-badge)](./6%20-%20Configurar%20Permissões.md)
<!-- [![Static Badge](https://img.shields.io/badge/7-Acesso_com_GITHUB-red?style=for-the-badge)](./7%20-%20Configurar%20GitHub%20ROSA.md) -->