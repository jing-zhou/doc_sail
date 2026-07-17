do this under ./certs 

Get 
===========================================

2 option :

A static efault certificate at  "test/certs/pebble.minica.pem"

B: sownload at: 
# 1) Skip proxy for 127.0.0.1 only
curl -vk --noproxy 127.0.0.1 --tlsv1.2 --http1.1 https://127.0.0.1:15000/roots/0  -o pebble-trust.pem

curl -vk --noproxy 127.0.0.1 --tlsv1.2 --http1.1 https://127.0.0.1:15000/intermediates/0 > pebble-intermediate.pem


# 2) Skip proxy for common local hosts
curl -vk --noproxy 'localhost,127.0.0.1,::1' https://127.0.0.1:15000/roots/0

# 3) Disable proxy for the single curl invocation
curl -vk --proxy "" https://127.0.0.1:15000/roots/0

# 4) Run curl without inheriting proxy env vars (bash)
env -u http_proxy -u https_proxy -u HTTP_PROXY -u HTTPS_PROXY curl -vk https://127.0.0.1:15000/roots/0

# 5) Persistently skip proxy in your shell
export NO_PROXY=localhost,127.0.0.1,::1
# (or set uppercase NO_PROXY/NO_PROXY depending on tools)

=========================================================
convert to java key store

keytool -import -alias pebble -file pebble-trust.pem -keystore pebble-truststore.jks -storepass changeit -noprompt




================================================

in the "pebble folder" do this to start pebble 

pebble -config ./test/config/pebble-config.json
====================================================
changes for openssl settings in order to use tcnative/openssl available for netty



===================================

disable pebbel once 

"rejectGoodNonces": 0, in config,json 

===================================================
start pebble with DNS ignore

pebble -dnsserver 127.0.0.1:8053 -config ./test/config/pebble-config.json
=================================
dsable DNS validation 

pebble -config test/config//pebble-config.json -strict=false
=====================================

pebble config: 

"pebble": {
    "dnsServer": "127.0.0.1:8053",
    ...
}
=========================================
install pebble-challtestsrv: 

go install ./cmd/pebble-challtestsrv

===========================================
start pebble-challtestsrv

pebble-challtestsrv -dns01="8053" --defaultIPv4="127.0.0.1"


===================================================
// dislay certs info at example.test:5001
openssl s_client -connect example.test:5001 -showcerts

=======================================================
//display pem file 
openssl x509 -in pebble.minica.pem -noout -subject -issuer -text | sed -n '1,35p'




