# 1. Create a VM 
- Login via SSH
# 2. Installation of HashiCorp Key Vault
```bash
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install vault
```
# 3. Run the Vault Server
```bash
vault server -dev -dev-listen-address="0.0.0.0:8200"
```
- Create the Secret Engine in the Vault UI.

- Take the new Terminal 
- Login to the VM via SSH
```bash
export VAULT_ADDR='http://0.0.0.0:8200'
```
# 4. Enable of App Role
```bash
vault auth enable approle
```

# 5. Creation of AppRole Policy
```bash
vault policy write terraform - <<EOF
path "*" {
  capabilities = ["list", "read"]
}

path "secrets/data/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "kv/data/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "kv/data/test-secret"{
  capabilities = ["read"]
}

path "secret/data/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "auth/token/create" {
capabilities = ["create", "read", "update", "list"]
}
EOF
```

# 6. Create an AppRole
```bash
    vault write auth/approle/role/terraform \
    secret_id_ttl=10m \
    token_num_uses=10 \
    token_ttl=20m \
    token_max_ttl=30m \
    secret_id_num_uses=40 \
    token_policies=terraform
```

# 7. Get RoleID and SecretID
- here in this case myPolicyName="Terraform"
1. RoleID
```bash
vault read auth/approle/role/<myPolicyName>/role-id
```
- regenerate secret ID for every 10 mins
2. SecretID
```bash
vault write -f auth/approle/role/<myPolicyName>/secret-id
```

# 8. Sample Terraform and Hashicorp Vault Key Value Configuration
```Hashicorp
provider "azurerm" {
  features {
    
  }
}

provider "vault" {
    address = "http://20.51.220.182:8200"
    skip_child_token = true
    auth_login {
      path = "auth/approle/login"
      parameters = {
        role_id="abc2a1f9-f375-199f-ae73-45bb2ea2fa84"
        secret_id="a57299a3-ec16-be39-f799-a214d9b4d26b"
      }
    }
}
data "vault_kv_secret_v2" "example" {
    # my secret is in path kv/data
    # secret:{"key"="your-secret-name", "value"="my-secret-value"}
  mount = "kv" // change it according to your mount
  name  = "data" 
  
}
variable "ssh_key_path" {
  type    = string
  default = "~/.ssh/id_ed25519.pub"
}
variable "rgName" {
  default = "sandeepRG"
}
variable "location"{
    default = "East US"
}
resource "azurerm_virtual_network" "exampleVnet"{
  name = "myVnet"
  location = var.location
  resource_group_name = var.rgName
  address_space = ["10.0.0.0/16"]

}
resource "azurerm_subnet" "internalSubnet" {
  name = "internal"
  virtual_network_name = azurerm_virtual_network.exampleVnet.name
  resource_group_name = var.rgName
  address_prefixes = ["10.0.2.0/24"]
}
resource "azurerm_public_ip" "examplePublicIP" {
  name = "myVMPulicIP1"
  resource_group_name = var.rgName
  location = var.location
  allocation_method = "Static"
}

resource "azurerm_network_interface" "exampleNetworkInterface" {
  name = "myNIC1"
  resource_group_name = var.rgName
  location = var.location

  ip_configuration {
    name = "internal"
    subnet_id = azurerm_subnet.internalSubnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id = azurerm_public_ip.examplePublicIP.id
  }
}
resource "azurerm_linux_virtual_machine" "exampleVM" {
  name = "myVM1"
  resource_group_name = var.rgName
  location = var.location
  network_interface_ids = [azurerm_network_interface.exampleNetworkInterface.id]
  size = "Standard_DC1s_v3"
  admin_ssh_key {
    username   = "adminuser"
    public_key = file(pathexpand(var.ssh_key_path))
  }
  admin_username = "adminuser"
  os_disk {
    caching = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }
  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2" # keep it gen2 for gen2 vm size
    version   = "latest"
  }
  tags = {
    secret=data.vault_kv_secret_v2.example.data["your-secret-name"]
  }
}
```

# AZURE Secret Engine Documentation
Documentation url: https://developer.hashicorp.com/vault/tutorials/secrets-management/azure-secrets
