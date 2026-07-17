# Let's Encrypt Certificate Manager for SOCKS5 Proxy Server

## Overview

This module provides automatic SSL/TLS certificate management using Let's Encrypt (ACME protocol) for your SOCKS5 proxy server. It handles certificate issuance, renewal, and hot-reloading automatically.

## Features

- ✅ **Automatic Certificate Issuance**: Obtains certificates from Let's Encrypt automatically
- ✅ **HTTP-01 Challenge**: Handles domain validation via HTTP-01 challenge
- ✅ **Auto-Renewal**: Automatically renews certificates before expiration
- ✅ **Multi-Domain Support**: Supports multiple domains in a single certificate (SAN)
- ✅ **Staging Environment**: Test with Let's Encrypt staging before production
- ✅ **REST API**: Manage certificates via HTTP API
- ✅ **PKCS12 Keystore**: Generates Java-compatible keystore files

## Components

### 1. LetsEncryptConfig
Configuration properties for Let's Encrypt integration.

### 2. LetsEncryptService
Core service that handles ACME protocol operations:
- Account registration
- Certificate ordering
- Domain authorization
- Certificate download
- Keystore generation

### 3. LetsEncryptRenewalScheduler
Scheduled task that automatically checks and renews certificates.

### 4. AcmeHttpChallengeHandler
HTTP endpoint handler for ACME HTTP-01 challenge validation.

### 5. LetsEncryptController
REST API for manual certificate management.

## Configuration

Add the following to your `application.properties`:

```properties
# Enable Let's Encrypt integration
letsencrypt.enabled=true

# Your email address (required for Let's Encrypt account)
letsencrypt.email=admin@example.com

# Domains to obtain certificates for (comma-separated)
letsencrypt.domains=example.com,www.example.com

# Use staging environment for testing (recommended for development)
letsencrypt.staging=true

# Certificate storage directory
letsencrypt.cert-directory=./certs

# HTTP challenge port (must be 80 for Let's Encrypt)
letsencrypt.challenge-port=80

# Keystore configuration
letsencrypt.key-store-password=changeit
letsencrypt.key-store-type=PKCS12
letsencrypt.key-alias=letsencrypt

# Renewal settings
letsencrypt.renewal-threshold-days=30
letsencrypt.renewal-check-interval-hours=24
```

## Usage

### Initial Setup

1. **Configure your domain DNS**: Point your domain(s) to your server's IP address

2. **Enable Let's Encrypt in configuration**:
   ```properties
   letsencrypt.enabled=true
   letsencrypt.email=your-email@example.com
   letsencrypt.domains=yourdomain.com
   letsencrypt.staging=true  # Start with staging
   ```

3. **Ensure port 80 is accessible**: Let's Encrypt requires HTTP port 80 for validation

4. **Start the server**: The certificate will be obtained automatically on startup

### Testing with Staging

Always test with Let's Encrypt staging environment first:

```properties
letsencrypt.staging=true
```

This prevents you from hitting Let's Encrypt rate limits during testing.

### Production Deployment

Once testing is successful, switch to production:

```properties
letsencrypt.staging=false
```

### Using the Certificate with SSL

Update your SSL configuration to use the Let's Encrypt certificate:

```properties
# Use Let's Encrypt certificate
server.ssl.key-store-type=PKCS12
server.ssl.key-store=file:./certs/letsencrypt.p12
server.ssl.key-store-password=changeit
server.ssl.key-alias=letsencrypt
```

## API Endpoints

### Get Certificate Status
```bash
curl http://localhost:8080/api/letsencrypt/status
```

Response:
```json
{
  "enabled": true,
  "domains": ["example.com"],
  "staging": false,
  "needsRenewal": false,
  "certificateInfo": {
    "subject": "CN=example.com",
    "issuer": "CN=Let's Encrypt Authority X3",
    "notBefore": "2024-01-01T00:00:00Z",
    "notAfter": "2024-04-01T00:00:00Z",
    "serialNumber": "abc123...",
    "domains": ["example.com"],
    "daysUntilExpiration": 60
  }
}
```

### Get Certificate Information
```bash
curl http://localhost:8080/api/letsencrypt/certificate
```

### Manual Renewal
```bash
curl -X POST http://localhost:8080/api/letsencrypt/renew
```

## File Structure

After successful certificate issuance, the following files are created:

```
./certs/
├── account.key          # ACME account private key
├── domain.key           # Domain private key
├── chain.crt            # Certificate chain (PEM format)
└── letsencrypt.p12      # PKCS12 keystore (for Java/Netty)
```

## Automatic Renewal

The scheduler automatically:
1. Checks certificate expiration daily (configurable)
2. Renews certificates 30 days before expiration (configurable)
3. Generates new keystore
4. Logs renewal status

## ACME HTTP-01 Challenge

The HTTP-01 challenge works as follows:

1. Let's Encrypt requests: `http://yourdomain.com/.well-known/acme-challenge/{token}`
2. Your server responds with the authorization string
3. Let's Encrypt verifies the response
4. Certificate is issued

The challenge handler is automatically registered and serves the tokens.

## Troubleshooting

### Certificate issuance fails

**Check DNS configuration**:
```bash
nslookup yourdomain.com
```

**Verify port 80 is accessible**:
```bash
curl http://yourdomain.com/.well-known/acme-challenge/test
```

**Check logs**:
```bash
tail -f logs/application.log | grep -i "letsencrypt"
```

### Rate Limits

Let's Encrypt has rate limits:
- 50 certificates per domain per week
- 5 failed validations per account per hour

**Solution**: Use staging environment for testing.

### Firewall Issues

Ensure port 80 is open:
```bash
sudo ufw allow 80/tcp
```

### Permission Issues

Ensure the application has write permissions to the certificate directory:
```bash
chmod 755 ./certs
```

## Security Considerations

1. **Protect Private Keys**: The `account.key` and `domain.key` files contain sensitive private keys
   ```bash
   chmod 600 ./certs/*.key
   ```

2. **Secure API Endpoints**: Consider adding authentication to the REST API endpoints

3. **HTTPS Redirect**: After obtaining certificates, redirect HTTP to HTTPS

4. **Backup**: Regularly backup your `./certs` directory

## Example: Complete Configuration

```properties
# Let's Encrypt Configuration
letsencrypt.enabled=true
letsencrypt.email=admin@example.com
letsencrypt.domains=proxy.example.com,www.proxy.example.com
letsencrypt.staging=false
letsencrypt.cert-directory=/etc/letsencrypt/certs
letsencrypt.key-store-password=MySecurePassword123!
letsencrypt.key-store-type=PKCS12
letsencrypt.key-alias=letsencrypt
letsencrypt.renewal-threshold-days=30
letsencrypt.renewal-check-interval-hours=24

# SSL Configuration (use Let's Encrypt certificate)
server.ssl.enabled=true
server.ssl.key-store-type=PKCS12
server.ssl.key-store=file:/etc/letsencrypt/certs/letsencrypt.p12
server.ssl.key-store-password=MySecurePassword123!
server.ssl.key-alias=letsencrypt
```

## Dependencies

The module uses the following libraries:
- `acme4j-client`: ACME protocol client
- `acme4j-utils`: Utilities for key generation and CSR creation

## License

This module is part of the SOCKS5 proxy server project.

## Support

For issues or questions:
1. Check the logs for detailed error messages
2. Verify your configuration
3. Test with Let's Encrypt staging first
4. Check Let's Encrypt status page: https://letsencrypt.status.io/

