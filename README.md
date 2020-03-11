# CDPMultinodeDocker

CDP Multinode script using Docker on Mac/Windows 10,
This will create brand new 6 instances on AWS( 2 4xlarge and 2 2xlarge)
CDP DC will be installed with full security (Kerberos,TLS and KMS)

Updated on March 11, 2020


Assumptions:

		1> This document assumes that you have access to an AWS account
		2> Partners or their IT Dept can create their own VPC, Subnet, key-pair and security group 
		in the same availability zone that will be used to create multi node instances in the script below.
		3> Request cloudera license from partner portal  
		4> Access to valid cloudera.com credentials to download binaries
		5> Access to the relevant script from partner portal here.
		6> Access to the following versions of docker are used for Mac OS and Windows 10 Pro. 
		https://hub.docker.com/editions/community/docker-ce-desktop-mac/
		https://hub.docker.com/editions/community/docker-ce-desktop-windows/

AWS Dependencies:

		1> AWS keypair (e.g. “.pem”) files to use with the scripts
		2> Decide on AWS region/AZ (us-east-1 used in this example)
		3> Ensure an equivalent CentOS image is available in your AZ,Example: ami-02eac2c0129f6376b #CentOS-7x86_64 
		4> Create a VPC(or use default), subnet and Security Group (SG) where these nodes are in the same AZ. 
		5> Record the SG to be used in the config files. Make sure the SG is open to all hosts in security group.
		
Download scripts,CDP DC bits and licence info:

		1> Download the scripts. Save the files to your home directory (e.g. Users/ssharma)
		   NOTE: For Windows, avoid using space in folder-names. 
		2> Copy the license file to this directory.You should have requested a trial license from the partner portal. 
		3> Copy the AWS  ".pem" file into the home directory (Users/ssharma)
		4> Create a directory say, mn-script. unzip the files here.  
		5> Create another directory under mn-script eg #mkdir bins
		6> Download the following or latest CSD’s into the “bins” directory 
		    https://archive.cloudera.com/CFM/csd/1.0.0.0/NIFI-1.9.0.1.0.0.0-90.jar 
		    https://archive.cloudera.com/CFM/csd/1.0.0.0/NIFICA-1.9.0.1.0.0.0-90.jar
		    https://archive.cloudera.com/CFM/csd/1.0.0.0/NIFIREGISTRY-0.3.0.1.0.0.0-90.jar
		Download the following or latest files into the “bins” directory
		CSP: https://www.cloudera.com/downloads/cdf/csp-trial.html(Version 0.8 - sha, parcel files and CSD)
		CSM: https://www.cloudera.com/downloads/cdf/csm-trial.html(Version 2.0 - sha, parcel files and CSD)

	NOTE: Make sure the SCHEMAREGISTRY, STREAMS_MESSAGING_MANAGER and STREAMS_REPLICATION_MANAGER files are 
	in the “bins” directory before executing the ansible playbook. 

Docker Setup:

	On both Windows and Mac OS, Following commands are used to setup the environment.
	We will execute the scripts to setup the 6-node cluster with all the relevant services. 
	Kerberos,KMS and TLS will be setup by default. 
	
	1> Ensure docker desktop has been installed and is running without any issues on your laptop. 
	2> Open a terminal on mac and command prompt on a windows machine. 
	   The set of instructions work on both Mac OS and Windows. 
	3> $docker run -it fedora /bin/bash, you will see docker id as example below. 
	      ...@077d2b4577cfb/mn-script#] exit;
	    Make a note the ID  "77d2b4577cfb" . Use this id to run the next command. 
	4> execute $docker commit 77d2b4577cfb  myfedora (Use the ID from command above)
	5>Mounting your local Mac drive  /Users/<dir> to Docker /home/<dir>
		
		Mac Example: $docker run -it --volume /Users/ssharma:/home/ssharma myfedora /bin/bash
		Windows Example: $docker run -it --volume C:\Users\ssharma:/home/ssharma myfedora /bin/bash
		
	6> At this time,you have a docker image with all the relevant files mapped to your home directory 
	eg: /home/ssharma.Next,we will prep the docker container and customize these files. 
        7> Install pyhton3 and boto3 in your Docker image 
	
		[root@2e3f9e83cf7a  ~]# dnf update -y
		[root@2e3f9e83cf7a  ~]# dnf install -y ansible python3-pip git  
		[root@2e3f9e83cf7a  ~]# pip3 install boto boto3
	
	8> Add SSH key on docker ( It is 2 step process )
	NOTE: On windows, you will need to copy the .pem file to a native docker folder and run these commands. 
	
	Step 1 : This step produces agent pid as below
	
	$[root@2e3f9e83cf7a  ~]#eval ‘ssh-agent -s’
	  SSH_AUTH_SOCK=/var/folders/3m/xs2m6r7x7_qg8wp11ggy8l000000gp/T//ssh-ASHkKOqJ6PpS/agent.51910; export SSH_AUTH_SOCK;
 		SSH_AGENT_PID=51911; export SSH_AGENT_PID;
       		echo Agent pid 51911;
		
	Step2: Use ssh-add command and provide pem file location 
 		
		$[root@2e3f9e83cf7a  ~] # ssh-add /home/ssharma/sunita_field.pem
  		Identity added: /home/ssharma/sunita_field.pem

      10> Adding key-vault : Create the ansible vault file in the root directory to store the private key. 
      Note:It will ask for password to create vault,We will store this in a password file as the next step
        
	 [root@2e3f9e83cf7a  ~]#ansible-vault create ssharma_keys.vault
       
      11> This will open up an editor similar to vi. Copy and paste your .pem contents,Pay close attention at the 		indentation.Give the key name and space for | , add 2 spaces for each line below key name 
      
	For Example: ssharma_keys.vault, give a <name>_key ex: sunita_key: | as shown below
	
	sunita_key: |
  	  -----BEGIN RSA PRIVATE KEY-----
  	  Madsfdasagafgfdgfdsgadhdjasvfgaertqrecsf
  	 [...]
  	 dfasdgretwreaqghaduogihafdkghareoighfdk=
  	 -----END RSA PRIVATE KEY-----
   
  	NOTE: Record the private key name (ex here is sunita_key) which will be used later in the config files
	
   	You will be asked to enter a password. Save the password. You can use this password in case you want to 
	view or edit the file at a later stage. Use ansible-vault view or ansible-vault edit to make changes
	   
		[root@2e3f9e83cf7a  ~]#ls -ltr /home/ssharma/ssharma_keys.vault (verify)
		
	12> On docker, let's now create a simple file to store the Vault password, so you won't be prompted at runtime,
	Create the file under your home directory
		[root@2e3f9e83cf7a  ~]#echo "YourPassword" > vault-password-file
		[root@2e3f9e83cf7a  ~]#chmod 400 vault-password-file
		
	NOTE: Record the file path and file name. We will use it in the config files

	13> On docker export variables for the AWS keys as below:
            
	    export AWS_ACCESS_KEY_ID=AKIAQxxxxxx
  	    export AWS_SECRET_ACCESS_KEY=uOI3N5KQZ8zbxxxxxxxxxx

Modify the configuration file:

At this point, you should have the script under a folder called mn-script.This folder should have the bin directory. 
We will also need access to the vault, pem and password files that are stored in the home directory.
The home directory should be accessible via docker mapping of the folders. 

	1>Open ../config/stock.infra.aws.yml file
	2>Make changes to parameters in stock.infra.aws.krb.yml where it says <replace me>. 
	eg Owner,project,enddate,vpc,region,subnet and security group.
	
	 region: us-east-1 <replace me>
	 subnet: subnet-76505a3cxx<replace me>
  	 security_group: sg-010c70ad828ad9axx<replace me>
         image: ami-02eac2c0129f6376b <replace me> # CentOS-7 x86_6
       
       tags:
        owner: user.test<replace me>
        enddate: "01312020"<replace me>
        project: ansible-test<replace me>
  
	3>Open and modify filepath for license in stock.cluster.krb.yml. where it says <replace me>
	Example:
	   licence:
   	   type: enterprise
      	   filepath: test_2019_2020_Licenseinfo/test_2019_2020_cloudera_license.txt<replace me>

	4>Open /etc/ansible/ansible.cfg  make the following changes and save.
	
		a> uncomment record_host_key
			# host key checking setting above.
			record_host_keys=False
			
		b> uncomment value_password_file and specify the location of your vault password file.
		 # specifying --vault-password-File on the command line.
		   vault_password_file = /home/ssharma/sunita-vault-password
	
	5>Open /etc/ansible/hosts, add following 2 lines as below and save:

	[local]
	localhost
	
	6>Change the following information in config/stock.cluster.krb.yml
	 a> Add the private_key value eg: {{ sunita_key }}
	 Example:
 	 # from vault file , replace <replace me> it with your own key  
  		private_key: "{{ replace_key }}"
		
         b> Provide the location for the parcel downloaded above from /bins folder (Step 6 Download and licence info)
	 where it says <replace me>
          Example: 
	     local_csds: 
   	           - /home/ssharma/bins/SCHEMAREGISTRY-0.8.0.jar <replace me>
  	           - /home/ssharma/bins/STREAMS_MESSAGING_MANAGER-2.1.0.jar<replace me>
	    local_parcels: 
 	    	- /home/ssharma/bins/SCHEMAREGISTRY-0.8.0.2.0.0.0-135-el7.parcel<replace me>>
      	     	- /home/ssharma/bins/SCHEMAREGISTRY-0.8.0.2.0.0.0-135-el7.parcel.sha<replace me>
      	     	- /home/ssharma/bins/STREAMS_MESSAGING_MANAGER-2.1.0.2.0.0.0-135-el7.parcel<replace me>
             	- /home/ssharma/bins/STREAMS_MESSAGING_MANAGER-2.1.0.2.0.0.0-135-el7.parcel.sha<replace me>

       
	7> For Auto_TLS, you will need a CDP DC license file from cloudera.
	Specify the path to that file as indicated below, whereever it says <replace me> 
 	
	 licence:
      	   type: enterprise
           filepath: test_2019_2020_Licenseinfo/test_2019_2020_cloudera_license.txt<replace me>
 

Now are ready to execute the ansible playbook from mn-script folder.

  $ansible-playbook site.yml -e "infra=config/stock.infra.aws.yml" -e "cluster=config/stock.cluster.krb.yml"  -e "vault= <path-to-keys.vault-file>" -e "cdpdc_teardown=<userid-date>" -e "public_key=<name_of_public_key_AWS>"

Example:

 ansible-playbook site.yml -e "infra=config/stock.infra.aws.yml" -e "cluster=config/stock.cluster.krb.yml"  -e "vault=/root/ashish_keys.vault" -e "cdpdc_teardown=sunita-03122020" -e "public_key=sunita-pse-sandbox"

After End of Successful Execution, You will see something like below as a Recap:

TASK [cdpdc_cm_server : reset var _api_command] ******************************************************************************************************************************
ok: [54.91.49.29]

PLAY RECAP ******************************************************************************************************************************
100.24.8.58                : ok=31   changed=17   unreachable=0    failed=0    skipped=0    rescued=0    ignored=1   
3.94.167.42                : ok=31   changed=17   unreachable=0    failed=0    skipped=0    rescued=0    ignored=1   
52.90.154.199              : ok=31   changed=17   unreachable=0    failed=0    skipped=0    rescued=0    ignored=1   
54.208.14.90               : ok=31   changed=17   unreachable=0    failed=0    skipped=0    rescued=0    ignored=1   
54.85.168.49               : ok=57   changed=37   unreachable=0    failed=0    skipped=0    rescued=0    ignored=1   
54.91.49.29                : ok=118  changed=56   unreachable=0    failed=0    skipped=2    rescued=0    ignored=1   

Use cm node ( 4xlarge ) to get into CM to verify the cluster status
above example shows [54.91.49.29] as cm server

https://54.91.49.29:7183/cmf/login
Pwd: admin/admin

Login into AWS, check AWS EC2 instance , you will be able to see following instances created   has 3 Worker nodes(2xlarge+100gb) , 1 CDF(2xlarge+100gb) ,1 CDSW(4xlarge+100gb) and 1(4xlarge+100gb) master nodes




	
