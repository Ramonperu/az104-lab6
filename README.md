# LAB 06 - Implement Traffic Management

Enero 25 de 2023*, Ramón Peinado Ruiz

Vamos a aprender a usar las identidades los administradores de trafico de azure

**Objetivos:**

------

• Task 1: Aprovisionar el laboratorio.

• Task 2: Configurar la topologia de red de concenrtador y radio.

• Task 3: Probar la transitividad del emperajemiento de redes virtuales.

• Task 4: Configurar el enrutamiento. 

• Task 5: Implementar el Load Balancer.

• Task 6: Implementar Application Gateway.



**Diagrama:**

------

<img src="/img/1ºimagenn.png" alt="2ºimagenn" style="zoom:80%;" />



### TASK 1:

------

Vamos a comenzar usando powershell, primeramente declararemos las variables del grupo y luego lo crearemos y desplegaremos las redes virtuales y cuatro maquinas virtuales mediante las plantillas ofrecidas por el profesor:

*$location = 'eastus'*

 *$rgName = 'az104-06-rg1'*

 *New-AzResourceGroup -Name $rgName -Location $location* 

-> Crea el grupo con los anteriores valores.



Hacemos el deployment de la plantilla (Anteriormente lo subimos a la platofarma)

*New-AzResourceGroupDeployment `*

 *-ResourceGroupName $rgName `  `* 

*-TemplateFile $HOME/az104-06-vms-loop-template.json `*   

*-TemplateParameterFile $HOME/az104-06-vms-loop-parameters.json `*



Ademas hemos de añadir el network watcher:

*$rgName = 'az104-06-rg1'* 

*$location = (Get-AzResourceGroup -ResourceGroupName $rgName).location $vmNames = (Get-AzVM -ResourceGroupName $rgName).Name* 

*foreach ($vmName in $vmNames) {*  

*Set-AzVMExtension `*

*-ResourceGroupName $rgName `*

*-Location $location `  -VMName $vmName `  `*

*-Name 'networkWatcherAgent' `* 

*-Publisher 'Microsoft.Azure.NetworkWatcher' `*

 *-Type 'NetworkWatcherAgentWindows' `*

 *-TypeHandlerVersion '1.4' }*

Nuestro grupo de recursos quedaria tal que así

<img src="/img/2ºimagenn.png" alt="2ºimagenn" style="zoom:80%;" />

### TASK 2:

------

Vamos a configurar el peering entre las redes virtuales anteriormente desplegadas

Hemos de anotar el Resource ID de la vnet2 y vnet3 dentro de sus propiedades.

Dentro de **vnet1>Settings>Peerings>Add**

Rellenamos con los siguientes datos

| Settings                                  | Value                                                        |
| ----------------------------------------- | ------------------------------------------------------------ |
| This virtual network: Peering link name   | **az104-06-vnet01_to_az104-06-vnet2**                        |
| Traffic to remote virtual network         | **Block traffic that originates from outside this virtual network** |
| Virtual network gateway                   | **None (default)**                                           |
| Remote virtual network: Peering link name | **az104-06-vnet2_to_az104-06-vnet01**                        |
| Virtual network deployment model          | ****Resource manager****                                     |
| I know my resource ID                     | enabled                                                      |
| resource ID                               | resource ID anteriormente anotada                            |

El resto de valores como vengan por defecto, primero hacemos la del 1 al 2 y viceversa y luego la del 1 al 3

<img src="/img/3ºimagenn.png" alt="3ºimagenn" style="zoom:80%;" />

### TASK 3:

------

Comprobación de la comunicación tras los emparejamientos mediante el Connection TroubleShoot Del Network Watcher:

<img src="/img/4ºimagenn.png" alt="3ºimagenn" style="zoom:80%;" />

El resultado es el mismo al hacerlo a la ip 10.62.0.4

Cuando lo hacemos de la maquina virtual 2 a las demas obtenemos el resultado de unreacheable

### TASK 4:

------

Configuramos el routing:

**vm0>az104-06-nic0>settings >IP configurations**

Activamos IP forwarding (Enabled)



Volvemos a la vm0 y seleccionamos **Run command**

*Install-WindowsFeature RemoteAccess -IncludeManagementTools*

Este comando instala el role de acceso remoto para windows server

Ejecutamos otro comando para el routing role service:

*Install-WindowsFeature -Name Routing -IncludeManagementTools -IncludeAllSubFeature* 

*Install-WindowsFeature -Name "RSAT-RemoteAccess-Powershell"* 

*Install-RemoteAccess -VpnType RoutingOnly* 

*Get-NetAdapter | Set-NetIPInterface -Forwarding Enabled*

Creamos las tablas de enrutamiento

**Services>Route Tables**

| Settings                 | Value             |
| ------------------------ | ----------------- |
| Subscription             | La enabled        |
| Resource group           | **az104-06-rg1**  |
| Location                 | eastus            |
| Name                     | **az104-06-rt23** |
| Propagate gateway routes | No                |

Accedemos a la tabla de rutas creada y añadimos la siguiente ruta

<img src="/img/5ºimagenn.png" alt="5ºimagenn" style="zoom:80%;" />

| Settings                             | Value                             |
| ------------------------------------ | --------------------------------- |
| Route name                           | **az104-06-route-vnet2-to-vnet3** |
| Address prefix destination           | **IP Addresses**                  |
| Destination IP addresses/CIDR ranges | **10.63.0.0/20**                  |
| Next hop type                        | **Virtual appliance**             |
| Next hop address                     | **10.60.0.4**                     |

Asociamos la subnet 0 de la vnet2 dentro del apartado subnets, queda justo debajo de rutas

| Settings        | Value              |
| --------------- | ------------------ |
| Virtual network | **az104-06-vnet2** |
| Subnet          | **subnet0**        |

Creamos otra tabla de enrutamiento esta vez de la 3 a la 2

| Settings                 | Value             |
| ------------------------ | ----------------- |
| Subscription             | La enabled        |
| Resource group           | **az104-06-rg1**  |
| Location                 | eastus            |
| Name                     | **az104-06-rt32** |
| Propagate gateway routes | No                |

Añadimos las rutas

| Settings                             | Value                             |
| ------------------------------------ | --------------------------------- |
| Route name                           | **az104-06-route-vnet3-to-vnet2** |
| Address prefix destination           | **IP Addresses**                  |
| Destination IP addresses/CIDR ranges | **10.62.0.0/20**                  |
| Next hop type                        | **Virtual appliance**             |
| Next hop address                     | **10.60.0.4**                     |

Asociamos la subnet de la red 3

| Settings        | Value              |
| --------------- | ------------------ |
| Virtual network | **az104-06-vnet3** |
| Subnet          | **subnet0**        |

<img src="/img/6ºimagenn.png" alt="5ºimagenn" style="zoom:80%;" />

<img src="/img/7ºimagenn.png" alt="5ºimagenn" style="zoom:80%;" />

Esta ultima comprobacion no no salia, el profesor nos recomendo dejar asi los peerings:

<img src="/img/8ºimagenn.png" alt="5ºimagenn" style="zoom:80%;" />

### TASK 5:

------

Creamos el balanceador de carga con los siguientes datos

| Settings       | Value                                                        |
| -------------- | ------------------------------------------------------------ |
| Subscription   | La enabled                                                   |
| Resource group | **az104-06-rg4**                                             |
| Name           | **az104-06-lb4**                                             |
| Region         | **name of the Azure region into which you deployed all other resources in this lab** |
| SKU            | Standard                                                     |
| Type           | **Public**                                                   |
| Tier           | **Regional**                                                 |

Frontend

| Settings          | Value             |
| ----------------- | ----------------- |
| Name              | **az104-06-pip4** |
| IP version        | IPv4              |
| IP type           | IP address        |
| Public IP address | **Create new**    |
| Availability zone | **No Zone**       |

Backend

| Settings                   | Value                     |
| -------------------------- | ------------------------- |
| Name                       | **** az104-06-lb4-be1**** |
| Virtual network            | **az104-06-vnet01**       |
| Backend Pool Configuration | **NIC**                   |
| IP Version                 | ****IPv4****              |

Add virtual machine; Añadimos ambas

Inbound rules> add load balancing route

| Settings                                           | Value                    |
| -------------------------------------------------- | ------------------------ |
| Name                                               | **az104-06-lb4-lbrule1** |
| IP Version                                         | **IPv4**                 |
| Frontend IP Address                                | **az104-06-pip4**        |
| Backend pool                                       | **az104-06-lb4-be1**     |
| Protocol                                           | **TCP**                  |
| Port                                               | 80                       |
| Backend port                                       | 80                       |
| Health probe                                       | **Create new**           |
| Name                                               | **az104-06-lb4-hp1**     |
| Protocol                                           | **TCP**                  |
| Port                                               | **80**                   |
| Interval                                           | **5**                    |
| Unhealthy threshold                                | **2**                    |
| Close the create health probe window               | **OK**                   |
| Session persistence                                | **None**                 |
| Idle timeout (minutes)                             | **4**                    |
| TCP reset                                          | **Disabled**             |
| Floating IP                                        | **Disabled**             |
| Outbound source network address translation (SNAT) | **Recommended**          |

Review +create > create

Vamos al recurso, guardamos la Frontend Ip configuration y comprobamos que el balanceo de carga funciona

<img src="/img/10ºimagenn.png" alt="5ºimagenn" style="zoom:80%;" />

<img src="/img/11ºimagenn.png" alt="5ºimagenn" style="zoom:80%;" />

### TASK 6:

------

Añadimos una subnet a la vnet1 con los siguientes datos

| Settings             | Value              |
| -------------------- | ------------------ |
| Name                 | **subnet-appgw**   |
| Subnet address range | **10.60.3.224/27** |

A continuacion buscamos en el portal Application Gateway > +create

Basics



| Settings                 | Value                             |
| ------------------------ | --------------------------------- |
| Subscription             | enabled                           |
| Resource group           | ****az104-06-rg5****              |
| Application gateway name | **az104-06-appgw5**               |
| Region                   | eastus                            |
| Tier                     | **Standard V2**                   |
| Enable autoscaling       | **No**                            |
| Instance count           | **2**                             |
| Availability zone        | **None**                          |
| HTTP2                    | **Disabled**                      |
| Virtual network          | **az104-06-vnet01**               |
| Subnet                   | **subnet-appgw (10.60.3.224/27)** |

Frontend

| Settings                 | Value                 |
| ------------------------ | --------------------- |
| Frontend IP address type | **Public**            |
| Public IP address        | **Add new**           |
| Name                     | ****az104-06-pip5**** |
| Availability zone        | **None**              |

Backend

| Settings                         | Value                   |
| -------------------------------- | ----------------------- |
| Name                             | **az104-06-appgw5-be1** |
| Add backend pool without targets | **No**                  |
| IP address or FQDN               | **10.62.0.4**           |
| IP address or FQDN               | **10.63.0.4**           |

Add routing rule

| Settings       | Value                   |
| -------------- | ----------------------- |
| Rule name      | **az104-06-appgw5-be1** |
| Priority       | **No**                  |
| Listener name  | **10.62.0.4**           |
| Frontend IP    | **10.63.0.4**           |
| Protocol       | **HTTP**                |
| Port           | **80**                  |
| Listener type  | **Basic**               |
| Error page url | **No**                  |

Backend targets

| Settings              | Value                         |
| --------------------- | ----------------------------- |
| Target type           | **Backend pool**              |
| Backend target        | **az104-06-appgw5-be1**       |
| Backend settings      | ****Add new****               |
| Backend settings name | ****az104-06-appgw5-http1**** |
| Backend protocol      | **HTTP**                      |
| Backend port          | **80**                        |
| Additional settings   | defaults                      |
| Host name             | defaults                      |

Creamos y hacemos lo mismo, vamos al application gateway y copiamos la ip publica del frontend

Hemos de ver como varia entre la vm3 y la vm2

<img src="/img/12ºimagenn.png" alt="5ºimagenn" style="zoom:80%;" />

<img src="/img/13ºimagenn.png" alt="13ºimagenn" style="zoom:80%;" />

Una vez hecha la comprobación podemos eliminar recursos.