# Enhanced Trojan Proxy Server - Documentation Index

## 📚 Complete Documentation Guide

This index helps you navigate the comprehensive documentation for the Enhanced Trojan Proxy Server with integrated Let's Encrypt certificate management.

---

## 🎯 Quick Start

**New to this project? Start here:**

1. **README_SIMPLE.md** ⭐ START HERE - Simple explanation of what this is
2. **PROJECT_FINAL_SUMMARY.md** - Complete project overview
3. **TROJAN_PROTOCOL_ARCHITECTURE.md** - Understanding the protocol
4. **LETSENCRYPT_QUICKSTART.md** - Getting Let's Encrypt running

---

## 🎭 Core Concept: Dual-Nature Disguise

**This server has two faces:**

### Face 1: HTTPS Web Server (Public) 🌐
When accessed with standard HTTPS:
- Serves "Welcome" page
- Handles Let's Encrypt validation
- Provides REST API
- **Looks completely normal**

### Face 2: SOCKS5 Proxy (Hidden) 🔒
When accessed with custom header:
- Full SOCKS5 proxy functionality
- TCP and UDP relay
- Only accessible with authentication
- **Hidden from outsiders**

**Same port. Same IP. Different behavior based on authentication.**

→ See **README_SIMPLE.md** for detailed explanation

---

## 📖 Core Documentation

### Protocol & Architecture

#### TROJAN_PROTOCOL_ARCHITECTURE.md (10K)
**What it covers:**
- Enhanced Trojan protocol specification
- Multiple encryption header support (30+ algorithms)
- Header format and structure
- Traffic flow diagrams
- Security analysis
- Comparison with standard Trojan
- DPI and traffic analysis resistance

**Read if you want to understand:**
- How the proxy protocol works
- Why it's resistant to detection
- Header format and validation
- Security model

#### PROJECT_FINAL_SUMMARY.md (9K)
**What it covers:**
- Complete project overview
- Architecture diagrams
- Component summary
- Deployment guide
- Configuration reference
- Testing instructions
- Advantages vs other proxies

**Read if you want to:**
- Understand the complete system
- Deploy to production
- Configure all components
- Test the implementation

---

### Let's Encrypt Integration

#### NETTY_MIGRATION_COMPLETE.md (9K)
**What it covers:**
- Integration approach (main HTTPS port)
- Why standalone port 80 was rejected
- RerouteHandler integration
- Architecture changes
- Component modifications
- Migration from Spring WebFlux to Netty

**Read if you want to understand:**
- Why Let's Encrypt is on port 2080, not port 80
- How ACME challenges work via HTTPS
- Integration architecture
- Technical implementation details

#### LETSENCRYPT.md (7K)
**What it covers:**
- Complete Let's Encrypt user guide
- Configuration properties reference
- File structure
- ACME HTTP-01 challenge explanation
- Troubleshooting guide
- Security considerations
- API reference

**Read if you want to:**
- Configure Let's Encrypt
- Understand ACME protocol
- Troubleshoot certificate issues
- Use the REST API

#### LETSENCRYPT_QUICKSTART.md (6K)
**What it covers:**
- Quick start guide
- Common commands
- Configuration checklist
- Testing procedures
- Common scenarios

**Read if you want to:**
- Get started quickly
- Run basic tests
- Use common commands
- Quick configuration reference

#### LETSENCRYPT_COMPLETE.md (5K)
**What it covers:**
- Implementation verification
- Statistics and metrics
- Component checklist
- Build status
- Test results

**Read if you want to:**
- Verify installation
- Check component status
- See implementation stats

#### LETSENCRYPT_NETTY_ARCHITECTURE.md (9K)
**What it covers:**
- Netty HTTP implementation details
- Request flow diagrams
- Pipeline configuration
- Threading model
- Performance comparison
- JSON serialization

**Read if you want to understand:**
- How Netty HTTP server works
- Request routing mechanism
- Performance characteristics
- Technical implementation

---

## 🔑 Authentication & User Management

### AUTH_API_GUIDE.md (18K) ⭐ NEW
**What it covers:**
- Complete REST API reference for authentication
- User registration, login, logout
- JWT token generation (3 alternatives)
- Workflow examples and use cases
- Error handling and troubleshooting
- cURL examples for testing
- Security best practices

**Read if you want to:**
- Implement client apps with authentication
- Understand the authentication API
- Generate and manage JWT tokens
- Build web-based user management UI
- Test authentication endpoints

### TOKEN_GENERATION_ALTERNATIVES.md (13K)
**What it covers:**
- 3 authentication alternatives in detail
- Alternative 1: Username + Password (for apps)
- Alternative 2: SessionId (for web browsers)
- Alternative 3: Current Token (periodic renewal)
- Periodic renewal pattern with health monitoring
- Self-connection loop prevention
- Client implementation examples

**Read if you want to:**
- Understand token generation methods
- Implement periodic token renewal in apps
- Build health monitoring for tokens
- Choose the right authentication method
- Avoid common pitfalls

### User Management Features
- User registration with unique username/email
- Login with username OR email
- Session management (24-hour sessions)
- JWT token generation with 3 alternatives
- Automatic token invalidation (one active token per user)
- Billing integration (automatic billing account creation)
- Password reset support (via email)

---

## 🔧 Setup & Configuration

### Initial Setup
```
1. Read: PROJECT_FINAL_SUMMARY.md (Deployment Guide section)
2. Read: LETSENCRYPT_QUICKSTART.md
3. Read: AUTH_API_GUIDE.md (if using authentication)
4. Configure: application.properties
5. Run: ./gradlew bootRun
```

### Configuration Files

#### application.properties
Main configuration file containing:
- Proxy settings (`params.*`)
- SSL/TLS settings (`server.ssl.*`)
- Let's Encrypt settings (`letsencrypt.*`)

Key settings:
```properties
# Main port (both SOCKS5 and HTTPS)
params.local-port=2080

# Secret for header authentication  
params.secret=your-password

# Let's Encrypt
letsencrypt.enabled=true
letsencrypt.email=admin@example.com
letsencrypt.domains=proxy.yourdomain.com
letsencrypt.staging=true  # Test first!
```

---

## 🧪 Testing

### Testing Order
1. **Build test**: `./gradlew compileJava`
2. **Welcome page**: `curl https://localhost:2080/`
3. **Let's Encrypt status**: `curl https://localhost:2080/api/letsencrypt/status`
4. **Certificate acquisition**: Check logs for ACME challenge
5. **Proxy test**: Use client with valid illiad header

### Test Documentation
- **LETSENCRYPT_QUICKSTART.md** - Testing section
- **PROJECT_FINAL_SUMMARY.md** - Testing section

---

## 🏗️ Architecture

### Understanding the Architecture

#### Layer 1: Network
```
Internet → Port 2080 (SSL/TLS)
```

#### Layer 2: Protocol Detection
```
SSL/TLS → HeaderDecoder
          ├─ Valid header → SOCKS5 Pipeline
          └─ Invalid header → HTTP Pipeline (Disguise)
```

#### Layer 3A: SOCKS5 Proxy
```
SOCKS5 Pipeline:
  V5ServerEncoder → V5CmdReqDecoder → V5CommandHandler
```

#### Layer 3B: HTTP Disguise
```
HTTP Pipeline:
  HttpServerCodec → HttpObjectAggregator → RerouteHandler
                                              ├─ /
                                              ├─ /.well-known/acme-challenge/*
                                              └─ /api/letsencrypt/*
```

### Architecture Documents
- **TROJAN_PROTOCOL_ARCHITECTURE.md** - Complete protocol architecture
- **PROJECT_FINAL_SUMMARY.md** - System architecture
- **NETTY_MIGRATION_COMPLETE.md** - Integration architecture

---

## 🔐 Security

### Security Features
1. **Multiple encryption algorithms** (30+ types)
2. **TLS encryption** (all traffic)
3. **HTTPS disguise** (fallback behavior)
4. **Secret verification** (header authentication)
5. **Traffic analysis resistance** (variable headers, HTTP fallback)
6. **DPI resistance** (no fixed patterns)

### Security Documentation
- **TROJAN_PROTOCOL_ARCHITECTURE.md** - Security analysis section
- **PROJECT_FINAL_SUMMARY.md** - Security model section

---

## 🚀 Deployment

### Production Deployment Checklist

#### Pre-deployment
- [ ] Read PROJECT_FINAL_SUMMARY.md deployment guide
- [ ] Configure DNS to point to server
- [ ] Set strong secret password
- [ ] Configure Let's Encrypt email
- [ ] Test with `letsencrypt.staging=true`

#### Deployment
- [ ] Build: `./gradlew build`
- [ ] Start: `./gradlew bootRun`
- [ ] Verify ACME challenge works
- [ ] Switch to `letsencrypt.staging=false`
- [ ] Update SSL config to use Let's Encrypt cert
- [ ] Restart server

#### Post-deployment
- [ ] Monitor logs
- [ ] Test proxy connectivity
- [ ] Verify certificate auto-renewal
- [ ] Test HTTP disguise
- [ ] Monitor API endpoints

---

## 📊 Components

### Core Components
| Component | File | Purpose |
|-----------|------|---------|
| Header Detection | HeaderDecoder.java | Validates encrypted headers |
| HTTP Disguise | RerouteHandler.java | Serves HTTP content, Let's Encrypt |
| SOCKS5 Encoder | V5ServerEncoder.java | Encodes SOCKS5 responses |
| SOCKS5 Decoder | V5CmdReqDecoder.java | Decodes SOCKS5 requests |
| SOCKS5 Handler | V5CommandHandler.java | Handles SOCKS5 commands |

### Let's Encrypt Components
| Component | File | Purpose |
|-----------|------|---------|
| Configuration | LetsEncryptConfig.java | Settings management |
| ACME Client | LetsEncryptService.java | ACME protocol implementation |
| Auto-renewal | LetsEncryptRenewalScheduler.java | Scheduled renewal |
| Certificate Model | CertificateInfo.java | Certificate data |

### Infrastructure
| Component | File | Purpose |
|-----------|------|---------|
| SSL Context | Ssl.java | SSL/TLS configuration |
| DTLS | Dtls.java | DTLS for UDP |
| Secret Manager | Secret.java | Header verification |
| Dependency Bus | ParamBus.java | Component injection |

---

## 🛠️ Development

### Code Structure
```
src/main/java/com/illiad/server/
├── codec/v5/           # SOCKS5 codecs
├── config/             # Configuration
├── handler/
│   ├── reroute/        # HTTP disguise (RerouteHandler)
│   ├── udp/            # UDP handling
│   └── v5/             # SOCKS5 handlers
└── security/
    ├── acme/           # Let's Encrypt components
    ├── Ssl.java        # SSL/TLS
    ├── Dtls.java       # DTLS
    └── Secret.java     # Authentication
```

### Build Commands
```bash
# Compile
./gradlew compileJava

# Build
./gradlew build

# Run
./gradlew bootRun

# Test
./gradlew test
```

---

## 🔍 Troubleshooting

### Common Issues

#### Certificate not obtained
→ Read: LETSENCRYPT.md (Troubleshooting section)

#### Port 2080 in use
→ Check: `netstat -tuln | grep 2080`

#### ACME challenge fails
→ Verify: DNS points to server, firewall allows 2080

#### Proxy not working
→ Check: Valid illiad header format and secret

### Troubleshooting Documentation
- **LETSENCRYPT.md** - Let's Encrypt troubleshooting
- **PROJECT_FINAL_SUMMARY.md** - General troubleshooting

---

## 📖 Reading Path

### For Operators
1. PROJECT_FINAL_SUMMARY.md
2. LETSENCRYPT_QUICKSTART.md
3. Deploy and configure

### For Developers  
1. TROJAN_PROTOCOL_ARCHITECTURE.md
2. NETTY_MIGRATION_COMPLETE.md
3. LETSENCRYPT_NETTY_ARCHITECTURE.md
4. Source code

### For Security Researchers
1. TROJAN_PROTOCOL_ARCHITECTURE.md (Security section)
2. PROJECT_FINAL_SUMMARY.md (Security model)
3. Source code analysis

---

## 📝 Summary

This project combines:
- ✅ Enhanced Trojan protocol (30+ encryption algorithms)
- ✅ SOCKS5 proxy
- ✅ HTTPS disguise
- ✅ Integrated Let's Encrypt
- ✅ Single port operation (2080)
- ✅ Traffic analysis resistance

**Everything documented. Everything integrated. Production ready.** 🎉

---

## 📄 Document Summary

| Document | Size | Category | Priority |
|----------|------|----------|----------|
| README_SIMPLE.md | 5K | Introduction | ⭐⭐⭐ START HERE |
| PROJECT_FINAL_SUMMARY.md | 9K | Overview | ⭐⭐⭐ Must Read |
| TROJAN_PROTOCOL_ARCHITECTURE.md | 10K | Protocol | ⭐⭐⭐ Must Read |
| LETSENCRYPT_QUICKSTART.md | 6K | Setup | ⭐⭐⭐ Must Read |
| NETTY_MIGRATION_COMPLETE.md | 9K | Integration | ⭐⭐ Technical |
| LETSENCRYPT.md | 7K | Reference | ⭐⭐ Reference |
| LETSENCRYPT_NETTY_ARCHITECTURE.md | 9K | Technical | ⭐ Deep Dive |
| LETSENCRYPT_COMPLETE.md | 5K | Verification | ⭐ Optional |

**Total Documentation: ~60K of comprehensive guides** 📚

