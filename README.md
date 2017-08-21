Thanks to beichhor (Benn Eichhorn) for his great contributions!

Hi all.  I wanted a way to dynamically create ebs-backed raid sets that were also growable.  My problem using CloudFormaiton templates with EBS devices was that I had to specify each one of the devices, mounts, and mount points.  This gets out of hand especially when testing multiple variations of RAID types and number of volumes.  I'm aware that this leaves the drives out of the control of CloudFormation during deletion, but at this point that is fine with me because I don't want to delete my EBS volumes anyhow.  Oh, and for those of you not using CloudFormation, this script works perfectly without it.



I set out to create a script that could run in UserData does just this.  Here's a simple example:

`raidformer.py --size 1 --count 2 --raidlevel 1 --mountpoint /nfsexports --wipe --tag foofoo --attach`

In this case, the script will:

* Allocate 2 EBS volumes with size of 1GB each.  
* Add a Tag of "Name=foofoo" to the volumes.
* Create the devices as /dev/sdf1 and /dev/sdf2.  
* Create a raid 1 device of /dev/md0
* Initialize /dev/md0 as a physical volume
* Create a volume group
* Create a logical volume
* Format the logical volume
* Add the fstab entry
* Mount the filesystem.

Alternatively, if you just wanted to create the raid device and add it to the VolumeGroup you could:

`raidformer.py --size 1 --count 2 --raidlevel 1  --tag foofoo --attach`

Or you could do a dry run of the commands and not allocate EBS volumes:

`raidformer.py --size 1 --count 2 --raidlevel 1 --mountpoint /nfsexports --wipe --tag foofoo --test`


If you just wanted to create & format the raid /logical volumes because you are already attaching volumes in your CloudFormation template: 

`raidformer.py --size 1 --count 2 --raidlevel 1 --mountpoint /nfsexports --wipe`

If you want to restore a raid configuration from a set of snapshots then provide a list of snapshot ids in the same order as they were originally mounted. Order preservation is essential to ensure snapshot is restored.

`raidformer.py --size=50 --raidlevel=10 --from-snapshot=snap-52bc5047,snap-52bc504f,snap-2bbc5051,snap-17bc5055 --filesystem=xfs --attach --mountpoint=/mnt/wowzers`

The script requires both a aws_access_key_id and  aws_secret_access_key set in your environment or in ~/.boto file.  The user needs the following IAM permissions:

	{
	  "Statement": [
	    {
	      "Sid": "Stmt1336836314319",
	      "Action": [
	        "ec2:AttachVolume",
	        "ec2:CreateTags",
	        "ec2:CreateVolume"
	      ],
	      "Effect": "Allow",
	      "Resource": [
	        "*"
	      ]
	    }
	  ]
	}

* This was tested on AMI: amzn-ami-pv-2012.03.1.x86_64-ebs (ami-e565ba8c)


Example usage in a CloudFormation template --  in my case I use an embedded puppet template for my configurations, so I've added the following parameters to the embedded template:

    "MountPoint" : {
      "Description" : "The mountpoint for the fileserver.",
      "Type" : "String",
      "Default" : "None"
    },
    "VolumeSize" : {
      "Description" : "The initial volume size.",
      "Type" : "Number",
      "Default" : "0"
    },

    "VolumeCount" : {
      "Description" : "The number of ebs volumes to add.",
      "Type" : "Number",
      "Default" : "0"
    },
    "RaidType" : {
      "Description" : "The raid level.",
      "Type" : "String",
      "Default" : "None",
      "AllowedValues": [
          "None", "0", "1","5", "10"
      ]
    },


and the following in the user data section:

            "RAID=", { "Ref" : "RaidType"}, "\n",
            "if [ $RAID != 'None' ]; then\n",
            "    /usr/local/sbin/ebs_raid.py --size ", { "Ref" : "VolumeSize"}, " --count ", { "Ref" : "VolumeCount"}, " --raidlevel ", { "Ref" : "RaidType"}, " --mountpoint ", { "Ref" : "MountPoint"}," --wipe --tag ", { "Ref" : "AWS::StackName" },"\n",
            "fi\n",
            "rm -f /root/.boto\n"


    "EBSSecretAccessKey": {
      "Default": "None",
      "Description" : "The Secret Access Key used for added EBS raid sets",
      "Type": "String"
    },
    "EBSAccessKeyID": {
      "Default": "None",
      "Description" : "The Access Key ID used for added EBS raid sets",
      "Type": "String"
    },


list of program options:


	Usage: raidformer.py [options]
	
	Options:
	  -h, --help            show this help message and exit
	  -a, --attach          Do the volume creation and attachment.
	  -c COUNT, --count=COUNT
	                        Number of EBS volumes
	  -d DEVICE, --device=DEVICE
	                        block device to start with
	  -f FILESYSTEM, --filesystem=FILESYSTEM
	                        filesystem type
	  -l LOGVOL, --logvol=LOGVOL
	                        Logical Volume Name
	  -m MOUNTPOINT, --mountpoint=MOUNTPOINT
	                        Mountpoint
	  --md=MD_DEVICE        md device name
	  -r RAIDLEVEL, --raidlevel=RAIDLEVEL
	                        RAID level
	  -s SIZE, --size=SIZE  Size of EBS volumes
	  --from-snapshot=SNAPSHOT_ID, --from-snapshot=SNAP1,SNAP2,...
	                        Create volumes from one or more ebs snapshots
	  -t, --test            Does a dry run of the mdadm lvm commands.
	  --tag=TAG             Tag name for the ebs devices
	  -v VOLGROUP, --volgroup=VOLGROUP
	                        Volume Group Name
	  -w, --wipe            format the new filesystem
