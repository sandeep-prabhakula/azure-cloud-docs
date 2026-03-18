# 1. Creation of VNet
- Create a VNET enabling the firewall and Bastion.
- Create a Firewall Policy.
- Create a subnet if required.

# 2. Creation of VMs
- Create the VM using the virtual network which is previously created and choose the subnet in which VM has to be running.
- Create multiple VMs if multiple subnets are being used.
- Provide as "Custom data" if there's any initialization script is required before running the application on the VM.
- Install required tools in the VMs and keep the app running.

# 3. Connect to Bastion
- username: azureuser
- Use SSH key .pem file to access the VM

# 4. Creating Firewall policy
- Create a DNAT Rule collection.
- Create a DNAT rule using the previously created rule collection.
- Choose Soure IP subnets(i.e choose which IPs can access the application via firewall) if it's open to internet keep SoucreIP as '*'.
- Choose Destination IP as the Firewall Public IP.
- Choose Translated IP as Private IP of frontend VM and port number in which the application is running.
- Note: always follow the same port numbers for serving inbound and outbound traffic for firewall and frontend VM

# 5. Install NGINX as webserver and Reverse Proxy for the application
- Configure nginx in the frontend VM and make sure not to expose backend VM IP or port numbers.
- update the default file with below config in etc/nginx/sites-default/default. 
```bash
    server {
        listen 80;
        root /var/www/html;
        index index.html index.htm index.nginx-debian.html;
        server_name _;
        location / {
            try_files $uri $uri/ =404;
        }
        location /api-mapping/{
            proxy_pass <backend vm private ip/requestMapping>; # example http://10.0.2.4:8080/api/
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
```
# 6. Create ASG for frontend VM and Backend VMS
- Create an ASG.
- Configure the ASG in VM's Networking Tab.

# 7. Configure ASG to Backend VM NSG.
- Create an Inbound rule 
    1. Source = Application security groups
        (i) Choose Frontend ASG
    2. Source Port Range = *
    3. Destination = Application security groups
            (i) Choose backend ASG
    4. Destination Port Range = 8080 # or the port in which backend app is running
    5. Protocol = Any
    6. Action = Allow #Keep Deny for the ports which are trying to access the application 
    7. Priority = 100
    8. Name = AnyName
    

