# Common Issues<a name="common-issues"></a>

The following are troubleshooting tips to help you solve common issues with EC2 instance running Windows Server\.

**Topics**
+ [EBS volumes don't initialize on Windows Server 2016 and later AMIs](#init-disks-win2k16)
+ [Boot an EC2 Windows Instance into Directory Services Restore Mode \(DSRM\)](#boot-dsrm)
+ [Unable to remotely log on to an instance with a user account that is not an Administrator](#remote-failure)

## EBS volumes don't initialize on Windows Server 2016 and later AMIs<a name="init-disks-win2k16"></a>

Instances created from Windows Server 2016 and later Amazon Machine Images \(AMIs\) use the EC2Launch service for a variety of startup tasks, including initializing EBS volumes\. By default, EC2Launch does not initialize secondary volumes\. You can configure EC2Launch to initialize these disks automatically\.

**To map drive letters to volumes**

1. Connect to the instance to configure and open the `C:\ProgramData\Amazon\EC2-Windows\Launch\Config\DriveLetterMappingConfig.json` file in a text editor\.

1. Specify the volume settings using the following format:

   ```
   {
     "driveLetterMapping": [
       {
         "volumeName": "sample volume",
         "driveLetter": "H"
       }
     ]
   }
   ```

1. Save your changes and close the file\.

1. Open Windows PowerShell and use the following command to run the EC2Launch script that initializes the disks:

   ```
   PS C:\>  C:\ProgramData\Amazon\EC2-Windows\Launch\Scripts\InitializeDisks.ps1
   ```

   To initialize the disks each time the instance boots, add the `-Schedule` flag as follows:

   ```
   PS C:\>  C:\ProgramData\Amazon\EC2-Windows\Launch\Scripts\InitializeDisks.ps1 -Schedule
   ```

## Boot an EC2 Windows Instance into Directory Services Restore Mode \(DSRM\)<a name="boot-dsrm"></a>

If an instance running Microsoft Active Directory experiences a system failure or other critical issues you can troubleshoot the instance by booting into a special version of Safe Mode called *Directory Services Restore Mode* \(DSRM\)\. In DSRM you can repair or recover Active Directory\.

### Driver Support for DSRM<a name="boot-dsrm-driver"></a>

How you enable DSRM and boot into the instance depends on the drivers the instance is running\. In the EC2 console you can view driver version details for an instance from the System Log\. The following tables shows which drivers are supported for DSRM\.


| Driver Versions | DSRM Supported? | Next Steps | 
| --- | --- | --- | 
| Citrix PV 5\.9 | No | Restore the instance from a backup\. You cannot enable DSRM\. | 
| AWS PV 7\.2\.0 | No | Though DSRM is not supported for this driver, you can still detach the root volume from the instance, take a snapshot of the volume or create an AMI from it, and attach it to another instance in the same availability zone as a secondary volume\. You can then enable DSRM \(as described in this section\)\. | 
| AWS PV 7\.2\.2 and later | Yes | Detach the root volume, attach it to another instance, and enable DSRM \(as described in this section\)\. | 
| Enhanced Networking | Yes | Detach the root volume, attach it to another instance, and enable DSRM \(as described in this section\)\. | 

For information about how to enable Enhanced Networking, see [Enabling Enhanced Networking on Windows Instances in a VPC](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/enhanced-networking.html)\. For more information about upgrading AWS PV drivers, see [Upgrading PV Drivers on Your Windows Instances](Upgrading_PV_drivers.md)\.

### Configure an Instance to Boot into DSRM<a name="configure-boot-dsrm"></a>

EC2 Windows instances do not have network connectivity before the operating system is running\. For this reason, you cannot press the F8 button on your keyboard to select a boot option\. You must use one of the following procedures to boot an EC2 Windows Server instance into DSRM\.

If you suspect that Active Directory has been corrupted and the instance is still running, you can configure the instance to boot into DSRM using either the System Configuration dialog box or the command prompt\.

**To boot an online instance into DSRM using the System Configuration dialog box**

1. In the **Run** dialog box, type `msconfig` and press Enter\.

1. Choose the **Boot** tab\.

1. Under **Boot options** choose **Safe boot**\.

1. Choose **Active Directory repair** and then choose **OK**\. The system prompts you to reboot the server\.

**To boot an online instance into DSRM using the command line**  
From a Command Prompt window, run the following command:

```
bcdedit /set safeboot dsrepair
```

If an instance is offline and unreachable, you must detach the root volume and attach it to another instance to enable DSRM mode\.

**To boot an offline instance into DSRM**

1. Open the Amazon EC2 console at [https://console\.aws\.amazon\.com/ec2/](https://console.aws.amazon.com/ec2/)\.

1. In the navigation pane, choose **Instances**\.

1. Locate the affected instance\. Open the context \(right\-click\) menu for the instance, choose **Instance State**, and then choose **Stop**\.

1. Choose **Launch Instance** and create a temporary instance in the same Availability Zone as the affected instance\. Choose an instance type that uses a different version of Windows\. For example, if your instance is Windows Server 2008 R1, then choose a Windows Server 2008 R2 instance\.
**Important**  
If you do not create the instance in the same Availability Zone as the affected instance you will not be able to attach the root volume of the affected instance to the new instance\.

1. In the navigation pane, choose **Volumes**\.

1. Locate the root volume of the affected instance\. [Detach](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/ebs-detaching-volume.html) the volume and [attach](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/ebs-attaching-volume.html) it to the temporary instance you created earlier\. Attach it with the default device name \(xvdf\)\.

1. Use Remote Desktop to connect to the temporary instance, and then use the Disk Management utility to [make the volume available for use](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/ebs-using-volumes.html)\.

1. Open a command prompt and run the following command\. Replace *D* with the actual drive letter of the secondary volume you just attached:

   ```
   bcdedit /store D:\Boot\BCD /set {default} safeboot dsrepair
   ```

1. In the Disk Management Utility, choose the drive you attached earlier, open the context \(right\-click\) menu, and choose **Offline**\.

1. In the EC2 console, detach the affected volume from the temporary instance and reattach it to your original instance with the device name `/dev/sda1`\. You must specify this device name to designate the volume as a root volume\.

1. [Start](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/Stop_Start.html) the instance\.

1. After the instance passes the health checks in the EC2 console, connect to the instance using Remote Desktop and verify that it boots into DSRM mode\.

1. \(Optional\) Delete or stop the temporary instance you created in this procedure\.

## Unable to remotely log on to an instance with a user account that is not an Administrator<a name="remote-failure"></a>

If you are not able to remotely log on to a Windows Server EC2 instance from a user account that is not an administrator account, ensure that you have granted the user the right to log on locally\. See [Grant a user or group the right to log on locally to the domain controllers in the domain](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-R2-and-2008/ee957044(v=ws.10)#grant-a-user-or-group-the-right-to-log-on-locally-to-the-domain-controllers-in-the-domain)\. 