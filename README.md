# Terraform Windows DNS Provider

This is the repository for a Terraform Windows DNS Provider, which you can use to create DNS records in Microsoft Windows DNS.  

This version of the provider creates corresponding PTR records for each DNS record with use of a Terraform module.

The provider uses the [github.com/gorillalabs/go-powershell/backend](github.com/gorillalabs/go-powershell/backend) package to "shell out" to PowerShell, fire up a WinRM session, and perform the actual DNS work. I made this decision because the Go WinRM packages I was able to find only supported WinRM in Basic/Unencrypted mode, which is not doable in our environment. Shelling out to PowerShell is admittedly ugly, but it allows the use of domain accounts, HTTPS, etc.

Create the following files: <br>
main.tf <br>
variables.tf <br>
./modules/add_host_and_pointer/main.tf <br>
./modules/add_host_and_pointer/variables.tf  <br>

# Using the Provider

### main.tf

```hcl
# Use the below examples as a template.
# If you do not specify a reverse_lookup_zone_name or pointer_name, a three octet reverse lookup zone and fourth octect pointer_name will be used by default.  
# Ex. 207.126.67.10 Defaults: reverse_lookup_zone_name = 67.126.207.in-addr.arpa, pointer_name = "10"

#TEMPLATE START
#Creates DNS Host (A) and Pointer (PTR) Records
module "TestEntry001" {
  source = "./modules/add_host_and_pointer"
  record_name = "TestEntry001"
  record_type = "A"
  zone_name = "TESTA.com"
  ipv4address = "10.0.0.2"
  reverse_lookup_zone_name = "0.0.10.in-addr.arpa"
  pointer_name = "2"
}

#Creates a CNAME Record
resource "windns" "TESTCNAME001" {
  record_name = "TESTCNAME001"
  record_type = "CNAME"
  zone_name = "TESTA.com"
  hostnamealias = "TESTCNAME001.TESTA.com"
}
#TEMPLATE END
```

### variables.tf 

```hcl
//# Configure the provider
//# username + password - used to build a powershell credential
//# adserver - the server we'll create a WinRM session into to perform the DNS operations
//# usessl - whether or not to use HTTPS for our WinRM session (by default port TCP/5986)
variable "adserver" {
  type        = "string"
  description = "Active Directory Server"
}

variable "username" {
  type        = "string"
  description = "Enter Username"
}

variable "password" {
  type        = "string"
  description = "Enter Password"
}

provider "windns" {
  server   = "${var.adserver}"
  username = "${var.username}"
  password = "${var.password}"
  usessl   = true
}
```

### ./modules/add_host_and_pointer/main.tf

```hcl
#Adds DNS (A) Records
resource "windns" "DNS" {
  record_name = "${var.record_name}"
  record_type = "A"
  zone_name   = "${var.zone_name}"
  ipv4address = "${var.ipv4address}"
}

#Adds Pointer (PTR) Records
resource "windns" "PTR" {
  record_name = "${var.record_name}.${var.zone_name}."
  record_type = "PTR"
  ipv4address = "${local.ptr_name}"
  zone_name   = "${local.rev_lookup_zone_name}"
}

#Local Variables
locals {
  octets                       = "${split(".",var.ipv4address)}"
  default_rev_lookup_zone_name = "${local.octets[2]}.${local.octets[1]}.${local.octets[0]}.in-addr.arpa"
  rev_lookup_zone_name         = "${var.reverse_lookup_zone_name == "" ? local.default_rev_lookup_zone_name : var.reverse_lookup_zone_name}"
  ptr_name                     = "${var.pointer_name == "" ? local.default_ptr_name : var.pointer_name}"
  default_ptr_name             = "${local.octets[3]}"
}

```

### ./modules/add_host_and_pointer/variables.tf

```hcl
variable "record_name" {
  type = "string"
}

variable "zone_name" {
  type = "string"
}

variable "ipv4address" {
  type = "string"
}

variable "pointer_name" {
  type    = "string"
  default = ""
}

variable "reverse_lookup_zone_name" {
  type    = "string"
  default = ""
}
```

# Building
0. Make sure you have $GOPATH set ($env:GOPATH='c:\wip\go' on Windows, etc)
1. go get github.com\portofportland\terraform-provider-windns
2. cd github.com\portofportland\terraform-provider-windns
3. go build
