## AWS Client VPN Setup Guide

Complete guide to set up AWS Client VPN for secure access to private AWS resources, similar to OpenVPN Access Server.

### Prerequisites

- AWS Account with appropriate permissions
- AWS CLI installed and configured
- Git installed
- Basic understanding of VPC networking

### Architecture Overview

```
Internet → AWS Client VPN Endpoint → Private Subnets → Private Resources
```

**VPC Configuration:**
- VPC CIDR: `10.0.0.0/16`
- Public Subnets: `10.0.1.0/24`, `10.0.2.0/24`
- Private Subnets: `10.0.10.0/24`, `10.0.20.0/24`
- Client VPN CIDR: `172.16.0.0/22` (non-overlapping)

### Step 1: Generate Certificates

AWS Client VPN requires mutual TLS authentication using certificates.

### Clone easy-rsa Repository

```bash
git clone https://github.com/OpenVPN/easy-rsa.git
cd easy-rsa/easyrsa3
```

### Initialize PKI and Create CA

```bash
# Initialize the PKI
./easyrsa init-pki

# Build Certificate Authority (CA)
./easyrsa build-ca nopass
```

When prompted, enter a Common Name for your CA (e.g., "VPN-CA").

### Generate Server Certificate

```bash
./easyrsa build-server-full server nopass
```
or 
```
./easyrsa build-server-full server.domain.tld nopass
```
### Generate Client Certificates

Create one certificate per user for better security and audit trails:

```bash
# Generate client certificates (one per user)
./easyrsa build-client-full client1 nopass
./easyrsa build-client-full client2 nopass
./easyrsa build-client-full client3 nopass
```

### Locate Generated Certificates

Your certificates are located in:
- **Server Certificate**: `pki/issued/server.crt`
- **Server Private Key**: `pki/private/server.key`
- **Client Certificates**: `pki/issued/client1.crt`, `pki/issued/client2.crt`, etc.
- **Client Private Keys**: `pki/private/client1.key`, `pki/private/client2.key`, etc.
- **CA Certificate**: `pki/ca.crt`

### Step 2: Import Certificates to AWS ACM

### Import Server Certificate

1. Go to **AWS Certificate Manager (ACM)** in your region
2. Click **Import certificate**
3. Fill in the fields:

**Certificate body:**
```bash
cat pki/issued/server.crt
# Copy the output including -----BEGIN CERTIFICATE----- and -----END CERTIFICATE-----
```

**Certificate private key:**
```bash
cat pki/private/server.key
# Copy the output
```

**Certificate chain:**
```bash
cat pki/ca.crt
# Copy the output
```

4. Click **Next** → **Import**

### Import Client Certificates

Repeat the above process for each client certificate:
- Use `pki/issued/client1.crt` as certificate body
- Use `pki/private/client1.key` as private key
- Use `pki/ca.crt` as certificate chain

**Note:** You need to import at least one client certificate. Import all client certificates you plan to use.

### Step 3: Create Client VPN Endpoint

1. Go to **VPC Console** → **Client VPN Endpoints** → **Create Client VPN Endpoint**

### Configuration Settings

| Setting | Value |
|---------|-------|
| Name tag | `test-vpn-endpoint` |
| Client IPv4 CIDR | `172.16.0.0/22` |
| Server certificate ARN | Select imported server cert |
| Authentication | Use mutual authentication |
| Client certificate ARN | Select imported client cert |
| Connection Logging | Optional (CloudWatch Logs) |
| DNS Server | `10.0.0.2` (VPC DNS) |
| Transport Protocol | UDP (recommended) |
| Split Tunnel | Enable |
| VPC | `test-vpc` |
| Security Group | Default VPC security group |

2. Click **Create Client VPN Endpoint**

### Step 4: Associate Target Networks

Associate your private subnets with the VPN endpoint:

1. Select your Client VPN Endpoint
2. Go to **Target network associations** tab
3. Click **Associate target network**
4. Select subnet: `test-pvtsn1` (10.0.10.0/24)
5. Click **Associate target network**
6. Repeat for `test-pvtsn2` (10.0.20.0/24)

Wait for status to become **Available**.

### Step 5: Add Authorization Rule

Allow VPN clients to access your VPC:

1. Go to **Authorization rules** tab
2. Click **Add authorization rule**
3. Configuration:
   - **Destination network**: `10.0.0.0/16` (your VPC CIDR) or `0.0.0.0/0`
   - **Grant access to**: All users
   - **Description**: Allow access to VPC
4. Click **Add authorization rule**

### Step 6: Add Route

Add routing to direct traffic to your VPC:

1. Go to **Route table** tab
2. Click **Create route**
3. Configuration:
   - **Route destination**: `10.0.0.0/16` or `0.0.0.0/0`
   - **Target VPC Subnet ID**: Select one of the associated subnets
   - **Description**: Route to VPC
4. Click **Create route**

### Step 7: Download Client Configuration

1. Select your Client VPN Endpoint
2. Click **Download client configuration**
3. Save the file as `downloaded-client-config.ovpn`

### Step 8: Prepare Client Configuration Files

For each user, you need to create a custom `.ovpn` file with their certificate embedded.

### Add Client Certificate and Key to Config

Edit the downloaded config file and add the following at the end:

```bash
# Add client certificate
'<cert>' >> downloaded-client-config.ovpn
#paste the content of the certs
cat pki/issued/client1.crt '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' >> downloaded-client-config.ovpn
'</cert>' >> downloaded-client-config.ovpn

# Add client private key
'<key>' >> downloaded-client-config.ovpn
#paste the content of the certs
cat pki/private/client1.key >> downloaded-client-config.ovpn
'</key>' >> downloaded-client-config.ovpn
```

Rename the file to `client1-config.ovpn` and distribute to the respective user.

Repeat for each client certificate (client2, client3, etc.).


### Step 9: Install and Connect

### Install AWS VPN Client

Download and install the AWS VPN Client:
- **Windows/Mac/Linux**: [AWS VPN Client Downloads](https://aws.amazon.com/vpn/client-vpn-download/)

### Connect to VPN

1. Open AWS VPN Client
2. Click **File** → **Manage Profiles** → **Add Profile**
3. Enter a profile name (e.g., "AWS Test VPN")
4. Select your `.ovpn` configuration file
5. Click **Add Profile**
6. Click **Connect**

### Verification

Once connected, verify access:

```bash
# Ping private EC2 instance
ping <private-ip-address>

# SSH to private instance
ssh -i your-key.pem ec2-user@<private-ip-address>

# Access RDS, ElastiCache, or other private services
```

### Troubleshooting

### Connection Failed

- Verify certificates are correctly imported in ACM
- Check Client VPN endpoint status is "Available"
- Ensure target networks are associated
- Verify authorization rules are configured

### Cannot Access Private Resources

- Check security groups allow traffic from `172.16.0.0/22`
- Verify route table has route to `10.0.0.0/16`
- Ensure split-tunnel is enabled if accessing internet simultaneously

### Certificate Errors

- Ensure client certificate is signed by the same CA as server certificate
- Verify certificate format includes BEGIN/END lines
- Check certificate hasn't expired

### Security Best Practices

1. **One certificate per user** - Never share certificates
2. **Enable connection logging** - Use CloudWatch for audit trails
3. **Use split-tunnel** - Only route AWS traffic through VPN
4. **Rotate certificates regularly** - Set expiration reminders
5. **Revoke compromised certificates** - Remove from ACM immediately
6. **Restrict authorization rules** - Use specific CIDRs instead of 0.0.0.0/0
7. **Use security groups** - Limit VPN client access to only required resources

### Certificate Revocation

To revoke a user's access:

1. Go to **ACM** → Select the client certificate
2. **Actions** → **Export certificate** (if needed for records)
3. **Actions** → **Delete**
4. User will lose VPN access within minutes

### Cost Considerations

AWS Client VPN pricing includes:
- **Endpoint association**: ~$0.10/hour per subnet association
- **Connection**: ~$0.05/hour per active connection
- **Data transfer**: Standard AWS data transfer rates

Estimated monthly cost for 5 users: ~$100-150/month

### Additional Resources

- [AWS Client VPN Documentation](https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/what-is.html)
- [OpenVPN easy-rsa Documentation](https://github.com/OpenVPN/easy-rsa)
- [AWS VPN Client User Guide](https://docs.aws.amazon.com/vpn/latest/clientvpn-user/client-vpn-user-what-is.html)
