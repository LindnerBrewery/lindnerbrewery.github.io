---
layout: post
title: Connecting VSCode to an AWS EC 2 Instance via SSM
tags: vscode aws ec2 
---
Lately, I've been working on a project that requires using AWS, and we have an EC2 instance that doesn't have port 22 open to the internet. Additionally, there's no VPN set up to access the VPC. Fortunately, AWS Systems Manager (SSM) allows us to access and manage our resources without exposing them to the internet or needing a VPN. While it's easy to get console access to an EC2 instance using SSM, writing code in a terminal isn't the most efficient. I wanted to leverage VSCode's Remote development feature to connect to my EC2 instance using SSM without exposing any ports or setting up a VPN. Here's how you can do it too.

## Step-by-Step Guide to Setting Up VSCode Remote-SSH with AWS SSM

### Step 1: Connect to EC2 with SSM

First, start a session with your EC2 instance using SSM:

``` console
aws ssm start-session --target i-07cedbf6cee69c2a0
```

You'll be logged in as ssm-user. Verify the current user with:

``` console
whoami
```

You should see:

```console
sh-5.2$ whoami 
ssm-user
```

### Step 2: Switch to ec 2 - user

Change to the `ec2-user`:

``` console
sudo su - ec2-user
```

Verify the switch by checking the user again:

```console
whoami
```

### Step 3: Create an SSH Key Pair (If You Haven't Already)

If you don't have an SSH key pair, create one using ssh-keygen.

### Step 4: Update authorized_keys

Navigate to your home directory:

```console
cd ~
```

Verify you're in the correct directory:

```console
pwd
```

You should see something like:

```console
[ec2-user@ip-10-0-122-4 ~]$ /home/ec2-user
```

Open the `authorized_keys file` in `vim`:

```console
vim ~/.ssh/authorized_keys
```

Add your public key to the `authorized_keys` file and save it.

### Step 5: Verify SSH Access

Exit the current SSM session and start a new SSM session with an SSH session:

```console
ssm start-session --target i-09cedbf6cie68c2b1 --profile default --region eu-central-1 --document-name AWSStartSSHSession --parameters portNumber=22
```

Ensure you have the correct region specified. Once the instance is running, open a new terminal and connect to the SSH endpoint:

```console
ssh ec2-user@ip-10-0-121-6.eu-central-1.compute.internal
```

You should see a message similar to this upon successful login:

```console
A newer release of "Amazon Linux" is available. 
	Version 2023.5.20240722: 
	Version 2023.5.20240730: 
Run "/usr/bin/dnf check-release-update" for full release and version update info 
	 , #_ 
	 ~\_ ####_         Amazon Linux 2023 
	~~  \_#####\ 
	~~     \###| 
	~~       \#/ ___   https://aws.amazon.com/linux/amazon-linux-2023 
	  ~~      V~' '-> 
	   ~~~        / 
	     ~~._.  _/ 
	       _/ _/ 
	     _/m/' 
Last login: Tue Aug 6 09:27:18 2024 
[ec2-user@ip-10-0-122-4 ~]$
```

### Step 6: Configure VSCode Remote SSH

Open the SSH Config file (~/.ssh/config) and add the following configuration:

```console
Host ip-10-0-121-6.eu-central-1.compute.internal 
	User ec2-user 
	IdentityFile ~/.ssh/id_rsa 
	ProxyCommand aws ssm start-session --target i-09cedbf6cie68c2b1 --profile default --region eu-central-1 -- document-name AWS-StartSSHSession --parameters portNumber=%p
```

The `ProxyCommand` starts the SSM session when initiating the SSH connection.

### Step 7: Start the Remote Session in VSCode

Now, open VSCode and start a new Remote SSH session. Wait for the VSCode server to be downloaded and started on the EC2 instance. Once that's done, you can code on your EC2 instance as if it were any other remote machine.

By following these steps, you can seamlessly connect to your EC2 instance using VSCode without exposing any ports or setting up a VPN.
Happy coding!