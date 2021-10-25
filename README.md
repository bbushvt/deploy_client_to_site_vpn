# HOW-TO deploy the Client-to-site VPN to connect to IBM Cloud Classic Resources
The process outlined below will walk through the deployment and configuration of the Client-to-site VPN in IBM Cloud to allow you to connect to IBM Cloud Classic resources
__This process is not endorsed by IBM, use at your own risk__

Setting up Client VPN on IBM Cloud
Prerequisites
- hostname for the VPN server (does not have to be registered in DNS)
- Client VPN IP Address range (needs to be a /22 or larger, must be unique from VPC prefixes and Classic subnets) Ex. 172.16.0.0/22
- Decide if full tunnel or split tunnel will be necessary
- List of subnets in Classic (these can be combined, ex if you have 10.10.10.0/24 and 10.10.11.0/24, you can make that 10.10.0.0/16)


1. Create a certificate manager 
	- login to cloud.ibm.com
	- click create resource
	- in the search field type Certificate Manager
	- click on the Certificate Manager tile to create that service
	- Give the service a name (something like Client VPN Certificate Manager)
	- the remaining options can be left to their defaults
	- click "I have read and agree to the following license agreements:" checkbox (lower right)
	- click "Create" (lower right)

2. Configure IAM access to the certificate manager from the Client-to-site Server
	- login to cloud.ibm.com
	- Click on Manage->Access (IAM) in the upper right hand corner
	- From the left hand menu, click on Authorizations
	- Click "Create +" on the right hand side
	- Select "VPC Infrastructure Servers" from the Source service drop down
	- Select the "Resources based on selected attributes" radio box
	- Check the "Resource type" check box
	- Select "Client VPN for VPC" from the drop down box
	- Select Certificate Manager from the "Target service" drop down box
	- Check the "Instance ID" check box
	- Select the name of the certificate manager created in step 1 from the "Instance ID" drop down box
	- Check "Writer" in the "Service access" section
	- Click Authorize to create the authorization

3. Download easy-rsa from https://github.com/OpenVPN/easy-rsa/releases
	- Select the right download for EasyRSA 3.0.8 (latest as of this writting)
		- Windows 32 bit - https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.8/EasyRSA-3.0.8-win32.zip
		- Windows 64 bit - https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.8/EasyRSA-3.0.8-win64.zip
		- Others (Mac/Linux/etc) - https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.8/EasyRSA-3.0.8.tgz
	- Extract (unzip/untar) the bundle 

4. Create your server certificates using easy-rsa
	- go into the easy-rsa directory that was created when unziping the bundle in step 3
	- Windows Procedure
		- Run EastRSA-Start.bat
		- This will start a shell session that can be used to create your certificates
		- follow the non-windows procedure for the rest of the commands
	- Non-windows Procedure
		* Initialize easy-rsa

			```
			./easyrsa init-pki
			```
		* Buid the certificate authority.  When running this command you will be asked for the "Common Name", you can use the hostname from the prerequisites section.
			```
			./easyrsa build-ca nopass
			```
		* Generate the server certificate
			```
			./easyrsa build-server-full <hostname from prerequisites section> nopass
			```

5. Import your certificates into the Certificate Manager created in step 1
	- login to cloud.ibm.com
	- In the resource summary section, click on "Services and software" (or click on "View all" and expand the "Services and software section")
	- Click on the Certificate Manager created in Step 1
	- On the right hand side, click "Import"
	- Enter a name to call this certificate, eg. Client VPN Certicate
	- Enter a description (optional)
	- For the certificate file, click Browse
		- Change the file type (lower right) from PEM File to All files
		- Go to the easy-rsa directory, then to the pki/issued directory
		- Select the crt file
	- For the Private ket file, click Browse
		- Go to the easy-rsa directory, then to the pki/private directory
		- select the key file that has the hostname from the prerequisites
	- For the Intermediate certificate file, click Browse
		- Change the file type (lower right) from PEM File to All files
		- Go to the easy-rsa directory, then to the pki directory (NOT the pki/private directory)
		- select the ca.key file
	- Click import

6. Create the VPC where the Client VPN will be deployed
	- login to cloud.ibm.com
	- in the upper left hand corner click on the 4 bar icon and click on VPC Infrastructure 
	- from the left hand menu, select VPCs (under network)
	- on the right hand side, click on "Create +"
	- Give your VPC a name
	- Select the region you would like to deploy the VPC (and thus the Client VPN)
	- Ensure that "Create a default prefix for each zone" is selected (this is the default and they should be ok to use)
	- Click "Create virtual private cloud" on the right hand side 

7. Modify the default Security group that was created to allow traffic on 443 through
	- login to cloud.ibm.com
	- in the upper left hand corner click on the 4 bar icon and click on VPC Infrastructure 
	- in the left hand menu, click on "Security groups"
	- Make sure the region selected is the region your VPC (from step 6) is deployed
	- Select the security group that is assigned to the VPC created in step 6
	- In the rules section, click on "Manage rules"
	- In the "Inbound rules" table, click on "Create +"
	- Set the protocol to TCP
	- make sure that "Port range" is selected
	- set both "Port min" and "Port max" to 443
	- Click "Save"

8. Deploy the Client VPN
	- login to cloud.ibm.com
	- in the upper left hand corner click on the 4 bar icon and click on VPC Infrastructure 
	- in the left hand menu, click on VPNs
	- Make sure that "Client-to-site servers Beta" is selected
	- Click on  "Create +" on the upper right hand of the VPN server table
	- Make sure that "Client-to-site server Beta" is selected as the server type
	- Enter a name for the client VPN server
	- Select the region that matches the VPC deployed in step 6
	- Select the VPC that was created in step 6
	- enter the client IP address pool from the Prerequisites
	- Choose either High-availability or Stand-alone mode
	- The default subnets should be fine
	- For the "Server certificate manager" select the certificate manager created in step 1
	- For the "Server SLL certificate" select the certificate created in step 5
	- uncheck "Client certificate" under "Client authentication"
	- check "User ID and passcode"
	- Make sure the Security group modified in step 7 is checked (this should be the default)
	- In "Additional configurations" make sure TCP is selected
	- Make sure that the correct tunnel mode (from Prerequisites) is selected
	- Click on "Create VPN server" on the right hand side (this will take several minutes to complete)

9. Deploy the Transit Gateway
	- login to cloud.ibm.com
	- Click on "Create resource"
	- Search for "Transit Gateway"
	- Click on the "Transit Gateway" tile
	- Enter the name of the transit gateway 
	- Ensure that "Local routing" is selected (this is the default)
	- Set the location to the geography where the VPC created in step 6 is located
	- In the Connection 1 box, make the following selections
		- Network connection should be VPC
		- Ensure that "Add new connection in this account" is selected (this is the default)
		- Select the region (should be only 1 because this is local)
		- In the "Available connections" drop down, select the VPC created in step 6
	- Click on "Add connection +"
	- In the Connection 2 box, make the following selections
		- Network connection should be "Classic infrastructure"
		- Ensure that "Add new connection in this account" is selected (this is the default)
		- Give this connection a name 
	- Click on "Create" on the right hand side

10. Configure Client-to-site VPN routing
	- login to cloud.ibm.com
	- in the upper left hand corner click on the 4 bar icon and click on VPC Infrastructure 
	- in the left hand menu, click on VPNs
	- Click on "Client-to-site servers Beta"
	- Click on the VPN we created in step 8
	- At the top, click on the "VPN server routes" section
	- For each subnet outlined in the Prerequisites:
		- Click on "Create +"
		- Enter a name (description) for this route
		- Put in the Destination CIDR address (ex 10.10.10.0/24)
		- Set the action to "Translate"
	- If you selected full tunnel and you would like to access the internet from the clients (via the tunnel) add a default route
		- Click on the "Create +"
		- Enter a name for this route (eg. Internet)
		- Enter 0.0.0.0/0 for the Destination CIDR
		- Set the action to "Translate"

11. Creating VPN Users IAM Group
	- login to cloud.ibm.com
	- click on "Manage" then "Access (IAM)" in the Upper right
	- click on "Access groups" on the left hand side
	- click on "Create +" on the right hand side
	- Give the access group a name (ex. "Client VPN Users")
	- Click on "Create"
	- Click on "Access policies"
	- Click the blue "Assign access +" button
	- Ensure the "IAM services" box is selected (it is by default)
	- From the "Which service do you wan to assign access to" drop down, select "VPC Infrastructure Services"
	- In the "Service access" section, check the "Users of the VPN server need this role to connect to the VPN server"
	- Click the "Add +" button at the bottom of the screen
	- Click the "Assign" button on the right hand side

12. Adding existing users to this group
	- login to cloud.ibm.com
	- click on "Manage" then "Access (IAM)" in the Upper right
	- click on "Access groups" on the left hand side
	- Select the group created in step 11
	- Ensure that the "Users" section is selected (default)
	- Click on "Add users +"
	- Check the box next to each existing user that should have Client-to-stite VPN access
	- Once you have selected all the users, click on "Add to group +"

13. Adding new users 
	- login to cloud.ibm.com
	- click on "Manage" then "Access (IAM)" in the Upper right
	- Click on "Invite Users +" in the upper right
	- Enter the email address of each user that needs to be invited in the "Enter email address" box (up to 100)
	- In the group table, click the "Add" link for the IAM Group that was created in step 11
	- Click the blue "Invite" button on the right and side
	- Each user will receive an email from IBM Cloud asking them to join an account in IBM Cloud
		- In that email there will be a "Join now" link they will need to click on
		- Following the link, they will need to enter their First Name, Last Name, and Password (country can be changed if needed)
		- Clicking "Join Account" will create their own IBMid User and add it to the correct account 

13. Configuring the Client systems
	- Download and install the correct OpenVPN software (more information here - https://cloud.ibm.com/docs/vpc?topic=vpc-client-to-site-vpn-planning&interface=ui)
		- Windows 8+ - https://openvpn.net/downloads/openvpn-connect-v3-windows.msi
		- Mac - https://openvpn.net/downloads/openvpn-connect-v3-macos.dmg
	- Give each client the profile for the VPN.  This should be done from the account that created the Client-to-site VPN, not the client accounts
		- login to cloud.ibm.com
		- Click on the 4 bar icon in the upper left hand corner and select "VPC Infrastructure"
		- On the left hand menu, click on VPNs
		- Click on the "Client-to-site servers" tab
		- Ensure that the "Region" dropdown is the region your Client-to-site VPN is deployed in
		- Click on the Client-to-site VPN created in step 8
		- Click on the "Download client profile" link to get the ".ovpn" profile file
	- On the client system, have them open the ovpn profile (double click on it on the windows platform)
	- When asked if they want to import the profile, click OK
	- For the username, enter the email address used for their IBMid
	- ensure that "Save password" is NOT checked
	- Before clicking connect, get the one time passcode using the following link
		- https://iam.cloud.ibm.com/identity/passcode
	- When ready to connect to the VPN, click "Connect"
	- Use the one time passcode as the Password
	- When the "Missing external certificate" warning appears, check the "Don't show again for this profile" checkbox and click "Continue"
	- You should not be connected to the Client-to-site VPN and will have access to all the routes setup in step 10