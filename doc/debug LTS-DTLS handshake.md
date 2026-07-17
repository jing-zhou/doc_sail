To debug TLS/DTLS handshake sessions in Java, you can use the built-in Java Secure Socket Extension (JSSE) debugging properties or external packet capture tools

1. Enable Built-in Java Debugging
The most common way is to use the javax.net.debug system property. This prints detailed logs of the handshake process, including protocol versions, cipher suites,
and certificate exchanges, directly to standard output.


    JVM Argument: Add this when starting your application:
        java -Djavax.net.debug=ssl:handshake:verbose -jar your-app.jar

    In-Code (Alternative): Set the property before the SSL context is initialized:
        System.setProperty("javax.net.debug", "ssl:handshake:verbose");

     Common Debugging Options:
         ssl:handshake: Shows each handshake message (ClientHello, ServerHello, etc.).
         verbose: Provides additional details for each handshake step.
         all: Enables all SSL/TLS debugging (includes record and session data), but can be extremely noisy.
         ssl:record: Useful for seeing the per-record tracing, including hex dumps of packets

 2. Analyze the Handshake Output
 When reviewing the logs, look for these specific "points of failure":
     Protocol Mismatch: Ensure both the client and server support the same version (e.g., TLS 1.2 or 1.3).

     Cipher Suite Issues: If no shared cipher suite is found, the handshake fails immediately after the ClientHello or ServerHello.

     Certificate Failures: Look for "fatal alert: certificate_unknown" or trust anchor issues. This often means the server's certificate is not in the client's truststore

 3. Use External Network Tools
 If Java's internal logs aren't enough (e.g., for DTLS timing issues), use network-level tools:
     Wireshark: Use the tls.handshake or dtls filter to see the raw packet exchange. You can see alerts like "Handshake Failure" or "Decrypt Error".

     OpenSSL: Test the connection manually to rule out Java-specific configuration issues:

     openssl s_client -connect host:port -debug -msg

 4. Advanced: SSL Traffic Decryption
 For deep debugging where you need to see the encrypted payload (post-handshake), tools like jSSLKeyLog can be used as a Java agent to export the (Pre)-Master-Secret to a file,
 which Wireshark can then use to decrypt the traffic.

    java -javaagent:jSSLKeyLog.jar -Djavax.net.ssl.keyStore=path/to/keystore -Djavax.net.ssl.keyStorePassword=password -jar your-app.jar

    Then, in Wireshark, set the (Pre)-Master-Secret log file to decrypt the TLS traffic.


5 config server/client to use ip v4 

    1. JVM Level (Recommended)
       The most reliable way to force Java to use IPv4 is through system properties. You can add these as VM arguments when starting your client and server applications:
       
       java -Djava.net.preferIPv4Stack=true -jar your-app.jar
            This forces the entire JVM to only use IPv4 and disable IPv6 completely.

       java -Djava.net.preferIPv4Addresses=true -jar your-app.jar
            This tells Java that if a hostname (like localhost) resolves to both IPv4 and IPv6, it should prioritize the IPv4 address.

    2. Application Level (Code)
        If you cannot change the startup arguments, set these properties at the very beginning of your main method before any network sockets are created:

        public static void main(String[] args) {
            System.setProperty("java.net.preferIPv4Stack", "true");
            // Ensure you use "127.0.0.1" instead of "localhost" to be explicit
            String serverIp = "127.0.0.1"; 

            // ... rest of your bootstrap code
}
