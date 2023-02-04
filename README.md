# Lab 05 - Implement Intersite Connectivity

# Student lab manual

## Rafael Román Martínez

## Enero 2023.

## Lab scenario

Contoso has its datacenters in Boston, New York, and Seattle offices connected via a mesh wide-area network links, with full connectivity between them. You need to implement a lab environment that will reflect the topology of the Contoso’s on-premises networks and verify its functionality.

**Note:** An **[interactive lab simulation](https://mslabs.cloudguides.com/guides/AZ-104 Exam Guide - Microsoft Azure Administrator Exercise 9)** is available that allows you to click through this lab at your own pace. You may find slight differences between the interactive simulation and the hosted lab, but the core concepts and ideas being demonstrated are the same.

## Objectives

In this lab, you will:

- Task 1: Provision the lab environment
- Task 2: Configure local and global virtual network peering
- Task 3: Test intersite connectivity

## Estimated timing: 30 minutes

## Architecture diagram

[![image](https://microsoftlearning.github.io/AZ-104-MicrosoftAzureAdministrator/Instructions/media/lab05.png)](https://microsoftlearning.github.io/AZ-104-MicrosoftAzureAdministrator/Instructions/media/lab05.png)

### Instructions

#### Task 1: Provision the lab environment

In this task, you will deploy three virtual machines, each into a separate virtual network, with two of them in the same Azure region and the third one in another Azure region.

1. Sign in to the [Azure portal](https://portal.azure.com/).

2. In the Azure portal, open the **Azure Cloud Shell** by clicking on the icon in the top right of the Azure Portal.

3. If prompted to select either **Bash** or **PowerShell**, select **PowerShell**.

   > **Note**: If this is the first time you are starting **Cloud Shell** and you are presented with the **You have no storage mounted** message, select the subscription you are using in this lab, and click **Create storage**.

4. In the toolbar of the Cloud Shell pane, click the **Upload/Download files** icon, in the drop-down menu, click **Upload** and upload the files **\Allfiles\Labs\05\az104-05-vnetvm-loop-template.json** and **\Allfiles\Labs\05\az104-05-vnetvm-loop-parameters.json** into the Cloud Shell home directory.

5. Edit the **Parameters** file you just uploaded and change the password. If you need help editing the file in the Shell please ask your instructor for assistance. As a best practice, secrets, like passwords, should be more securely stored in the Key Vault.

6. From the Cloud Shell pane, run the following to create the resource group that will be hosting the lab environment. The first two virtual networks and a pair of virtual machines will be deployed in [Azure_region_1]. The third virtual network and the third virtual machine will be deployed in the same resource group but another [Azure_region_2]. (replace the [Azure_region_1] and [Azure_region_2] placeholder, including the square brackets, with the names of two different Azure regions where you intend to deploy these Azure virtual machines. An example is $location1 = ‘eastus’. You can use Get-AzLocation to list all locations.):

   CodeCopy

   ```powershell
   $location1 = 'eastus'
   
   $location2 = 'westus'
   
   $rgName = 'az104-05-rg1'
   
   New-AzResourceGroup -Name $rgName -Location $location1
   ```

   > **Note**: The regions used above were tested and known to work when this lab was last officially reviewed. If you would prefer to use different locations, or they no longer work, you will need to identify two different regions that Standard D2Sv3 virtual machines can be deployed into.
   >
   > In order to identify Azure regions, from a PowerShell session in Cloud Shell, run **(Get-AzLocation).Location**
   >
   > Once you have identified two regions you would like to use, run the command below in the Cloud Shell for each region to confirm that you can deploy Standard D2Sv3 virtual machines
   >
   > `az vm list-skus --location <Replace with your location> -o table --query "[? contains(name,'Standard_D2s')].name"`
   >
   > If the command returns no results, then you need to choose another region. Once you have identified two suitable regions, you can adjust the regions in the code block above.

7. From the Cloud Shell pane, run the following to create the three virtual networks and deploy virtual machines into them by using the template and parameter files you uploaded:

   CodeCopy

   ```powershell
   New-AzResourceGroupDeployment `
      -ResourceGroupName $rgName `
      -TemplateFile $HOME/az104-05-vnetvm-loop-template.json `
      -TemplateParameterFile $HOME/az104-05-vnetvm-loop-parameters.json `
      -location1 $location1 `
      -location2 $location2
   ```

   > **Note**: Wait for the deployment to complete before proceeding to the next step. This should take about 2 minutes.

8. Close the Cloud Shell pane.

#### Task 2: Configure local and global virtual network peering

In this task, you will configure local and global peering between the virtual networks you deployed in the previous tasks.

1. In the Azure portal, search for and select **Virtual networks**.

2. Review the virtual networks you created in the previous task and verify that the first two are located in the same Azure region and the third one in a different Azure region.

   > **Note**: The template you used for deployment of the three virtual networks ensures that the IP address ranges of the three virtual networks do not overlap.

3. In the list of virtual networks, click **az104-05-vnet0**.

4. On the **az104-05-vnet0** virtual network blade, in the **Settings** section, click **Peerings** and then click **+ Add**.

5. Add a peering with the following settings (leave others with their default values) and click **Add**:

   | Setting                                                      | Value                                                        |
   | :----------------------------------------------------------- | :----------------------------------------------------------- |
   | This virtual network: Peering link name                      | **az104-05-vnet0_to_az104-05-vnet1**                         |
   | This virtual network: Traffic to remote virtual network      | **Allow (default)**                                          |
   | This virtual network: Traffic forwarded from remote virtual network | **Block traffic that originates from outside this virtual network** |
   | Virtual network gateway                                      | **None**                                                     |
   | Remote virtual network: Peering link name                    | **az104-05-vnet1_to_az104-05-vnet0**                         |
   | Virtual network deployment model                             | **Resource manager**                                         |
   | I know my resource ID                                        | unselected                                                   |
   | Subscription                                                 | the name of the Azure subscription you are using in this lab |
   | Virtual network                                              | **az104-05-vnet1**                                           |
   | Traffic to remote virtual network                            | **Allow (default)**                                          |
   | Traffic forwarded from remote virtual network                | **Block traffic that originates from outside this virtual network** |
   | Virtual network gateway                                      | **None**                                                     |

   > **Note**: This step establishes two local peerings - one from az104-05-vnet0 to az104-05-vnet1 and the other from az104-05-vnet1 to az104-05-vnet0.

   > **Note**: In case you run into an issue with the Azure portal interface not displaying the virtual networks created in the previous task, you can configure peering by running the following PowerShell commands from Cloud Shell:

   CodeCopy

   ```powershell
   $rgName = 'az104-05-rg1'
   
   $vnet0 = Get-AzVirtualNetwork -Name 'az104-05-vnet0' -ResourceGroupName $rgname
   
   $vnet1 = Get-AzVirtualNetwork -Name 'az104-05-vnet1' -ResourceGroupName $rgname
   
   Add-AzVirtualNetworkPeering -Name 'az104-05-vnet0_to_az104-05-vnet1' -VirtualNetwork $vnet0 -RemoteVirtualNetworkId $vnet1.Id
   
   Add-AzVirtualNetworkPeering -Name 'az104-05-vnet1_to_az104-05-vnet0' -VirtualNetwork $vnet1 -RemoteVirtualNetworkId $vnet0.Id
   ```

6. On the **az104-05-vnet0** virtual network blade, in the **Settings** section, click **Peerings** and then click **+ Add**.

7. Add a peering with the following settings (leave others with their default values) and click **Add**:

   | Setting                                                      | Value                                                        |
   | :----------------------------------------------------------- | :----------------------------------------------------------- |
   | This virtual network: Peering link name                      | **az104-05-vnet0_to_az104-05-vnet2**                         |
   | This virtual network: Traffic to remote virtual network      | **Allow (default)**                                          |
   | This virtual network: Traffic forwarded from remote virtual network | **Block traffic that originates from outside this virtual network** |
   | Virtual network gateway                                      | **None**                                                     |
   | Remote virtual network: Peering link name                    | **az104-05-vnet2_to_az104-05-vnet0**                         |
   | Virtual network deployment model                             | **Resource manager**                                         |
   | I know my resource ID                                        | unselected                                                   |
   | Subscription                                                 | the name of the Azure subscription you are using in this lab |
   | Virtual network                                              | **az104-05-vnet2**                                           |
   | Traffic to remote virtual network                            | **Allow (default)**                                          |
   | Traffic forwarded from remote virtual network                | **Block traffic that originates from outside this virtual network** |
   | Virtual network gateway                                      | **None**                                                     |

   > **Note**: This step establishes two global peerings - one from az104-05-vnet0 to az104-05-vnet2 and the other from az104-05-vnet2 to az104-05-vnet0.

   > **Note**: In case you run into an issue with the Azure portal interface not displaying the virtual networks created in the previous task, you can configure peering by running the following PowerShell commands from Cloud Shell:

   CodeCopy

   ```powershell
   $rgName = 'az104-05-rg1'
   
   $vnet0 = Get-AzVirtualNetwork -Name 'az104-05-vnet0' -ResourceGroupName $rgname
   
   $vnet2 = Get-AzVirtualNetwork -Name 'az104-05-vnet2' -ResourceGroupName $rgname
   
   Add-AzVirtualNetworkPeering -Name 'az104-05-vnet0_to_az104-05-vnet2' -VirtualNetwork $vnet0 -RemoteVirtualNetworkId $vnet2.Id
   
   Add-AzVirtualNetworkPeering -Name 'az104-05-vnet2_to_az104-05-vnet0' -VirtualNetwork $vnet2 -RemoteVirtualNetworkId $vnet0.Id
   ```

8. Navigate back to the **Virtual networks** blade and, in the list of virtual networks, click **az104-05-vnet1**.

9. On the **az104-05-vnet1** virtual network blade, in the **Settings** section, click **Peerings** and then click **+ Add**.

10. Add a peering with the following settings (leave others with their default values) and click **Add**:

    | Setting                                                      | Value                                                        |
    | :----------------------------------------------------------- | :----------------------------------------------------------- |
    | This virtual network: Peering link name                      | **az104-05-vnet1_to_az104-05-vnet2**                         |
    | This virtual network: Traffic to remote virtual network      | **Allow (default)**                                          |
    | This virtual network: Traffic forwarded from remote virtual network | **Block traffic that originates from outside this virtual network** |
    | Virtual network gateway                                      | **None**                                                     |
    | Remote virtual network: Peering link name                    | **az104-05-vnet2_to_az104-05-vnet1**                         |
    | Virtual network deployment model                             | **Resource manager**                                         |
    | I know my resource ID                                        | unselected                                                   |
    | Subscription                                                 | the name of the Azure subscription you are using in this lab |
    | Virtual network                                              | **az104-05-vnet2**                                           |
    | Traffic to remote virtual network                            | **Allow (default)**                                          |
    | Traffic forwarded from remote virtual network                | **Block traffic that originates from outside this virtual network** |
    | Virtual network gateway                                      | **None**                                                     |

    > **Note**: This step establishes two global peerings - one from az104-05-vnet1 to az104-05-vnet2 and the other from az104-05-vnet2 to az104-05-vnet1.

    > **Note**: In case you run into an issue with the Azure portal interface not displaying the virtual networks created in the previous task, you can configure peering by running the following PowerShell commands from Cloud Shell:

    CodeCopy

    ```powershell
    $rgName = 'az104-05-rg1'
    
    $vnet1 = Get-AzVirtualNetwork -Name 'az104-05-vnet1' -ResourceGroupName $rgname
    
    $vnet2 = Get-AzVirtualNetwork -Name 'az104-05-vnet2' -ResourceGroupName $rgname
    
    Add-AzVirtualNetworkPeering -Name 'az104-05-vnet1_to_az104-05-vnet2' -VirtualNetwork $vnet1 -RemoteVirtualNetworkId $vnet2.Id
    
    Add-AzVirtualNetworkPeering -Name 'az104-05-vnet2_to_az104-05-vnet1' -VirtualNetwork $vnet2 -RemoteVirtualNetworkId $vnet1.Id
    ```

#### Task 3: Test intersite connectivity

In this task, you will test connectivity between virtual machines on the three virtual networks that you connected via local and global peering in the previous task.

1. In the Azure portal, search for and select **Virtual machines**.

2. In the list of virtual machines, click **az104-05-vm0**.

3. On the **az104-05-vm0** blade, click **Connect**, in the drop-down menu, click **RDP**, on the **Connect with RDP** blade, click **Download RDP File** and follow the prompts to start the Remote Desktop session.

   > **Note**: This step refers to connecting via Remote Desktop from a Windows computer. On a Mac, you can use Remote Desktop Client from the Mac App Store and on Linux computers you can use an open source RDP client software.

   > **Note**: You can ignore any warning prompts when connecting to the target virtual machines.

4. When prompted, sign in by using the **Student** username and the password from your parameters file.

5. Within the Remote Desktop session to **az104-05-vm0**, right-click the **Start** button and, in the right-click menu, click **Windows PowerShell (Admin)**.

6. In the Windows PowerShell console window, run the following to test connectivity to **az104-05-vm1** (which has the private IP address of **10.51.0.4**) over TCP port 3389:

   CodeCopy

   ```powershell
   Test-NetConnection -ComputerName 10.51.0.4 -Port 3389 -InformationLevel 'Detailed'
   ```

   > **Note**: The test uses TCP 3389 since this is this port is allowed by default by operating system firewall.

7. Examine the output of the command and verify that the connection was successful.

8. In the Windows PowerShell console window, run the following to test connectivity to **az104-05-vm2** (which has the private IP address of **10.52.0.4**):

   CodeCopy

   ```powershell
   Test-NetConnection -ComputerName 10.52.0.4 -Port 3389 -InformationLevel 'Detailed'
   ```

9. Switch back to the Azure portal on your lab computer and navigate back to the **Virtual machines** blade.

10. In the list of virtual machines, click **az104-05-vm1**.

11. On the **az104-05-vm1** blade, click **Connect**, in the drop-down menu, click **RDP**, on the **Connect with RDP** blade, click **Download RDP File** and follow the prompts to start the Remote Desktop session.

    > **Note**: This step refers to connecting via Remote Desktop from a Windows computer. On a Mac, you can use Remote Desktop Client from the Mac App Store and on Linux computers you can use an open source RDP client software.

    > **Note**: You can ignore any warning prompts when connecting to the target virtual machines.

12. When prompted, sign in by using the **Student** username and the password from your parameters file.

13. Within the Remote Desktop session to **az104-05-vm1**, right-click the **Start** button and, in the right-click menu, click **Windows PowerShell (Admin)**.

14. In the Windows PowerShell console window, run the following to test connectivity to **az104-05-vm2** (which has the private IP address of **10.52.0.4**) over TCP port 3389:

    CodeCopy

    ```powershell
    Test-NetConnection -ComputerName 10.52.0.4 -Port 3389 -InformationLevel 'Detailed'
    ```

    > **Note**: The test uses TCP 3389 since this is this port is allowed by default by operating system firewall.

15. Examine the output of the command and verify that the connection was successful.

#### Clean up resources