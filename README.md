# task3_scenario3

Kafka Cluster Authentication Issue and Solution

Problem Statement:

The client, using the superuser bob with the password bob-secret, is unable to interact with certain Kafka brokers (kafka2 and kafka3) when using the following Kafka tools:

    kafka-topics
    kafka-console-producer
    kafka-console-consumer

Although the kafka1 broker works correctly as the bootstrap server, an error occurs when trying to use kafka2 or kafka3 as the bootstrap server. The following error message is encountered:

    [2023-07-26 13:20:58,720] ERROR [AdminClient clientId=adminclient-1] Connection to node 3 (kafka3/192.168.112.8:39092) failed authentication due to: Authentication failed: Invalid username or password (org.apache.kafka.clients.NetworkClient)
    [2023-07-26 13:20:58,722] WARN [AdminClient clientId=adminclient-1] Metadata update failed due to authentication error (org.apache.kafka.clients.admin.internals.AdminMetadataManager)
    org.apache.kafka.common.errors.SaslAuthenticationException: Authentication failed: Invalid username or password
    Error while executing topic command : Authentication failed: Invalid username or password


Root Cause

    The issue was traced to incorrect SSL configuration and a misconfiguration in the server.properties file on the kafka2 and kafka3 brokers. The password for the superuser bob was incorrectly set in the server configuration files.

Incorrect line in server.properties for kafka2 and kafka3:

    user_bob="b0b-secret"

Correct line:

    user_bob="bob-secret"

This typo caused the Kafka clients to fail authentication, leading to the "Invalid username or password" error.


Solution:

Step 1: Configure SSL for Kafka Brokers

    1.1. Create a CA (Certificate Authority)

    First, create a new CA key and certificate:

        openssl req -new -x509 -days 365 -keyout ca-key -out ca-cert -subj "/C=DE/ST=NRW/L=MS/O=juplo/OU=kafka/CN=Root-CA" -passout pass:kafka-broker

    1.2. Create a Truststore and Import the CA Certificate

    For each Kafka broker, create a truststore and import the CA certificate into it:

        keytool -keystore kafka.server.truststore.jks -storepass kafka-broker -import -alias ca-root -file ca-cert -noprompt

    1.3. Create a Keystore and Private Key

    Generate a new keystore and private key for each Kafka broker:

        keytool -keystore kafka.server.keystore.jks -storepass kafka-broker -alias kafka1 -validity 365 -keyalg RSA -genkeypair -keypass kafka-broker -dname "CN=kafka1,OU=kafka,O=juplo,L=MS,ST=NRW,C=DE"

    1.4. Generate a Certificate Signing Request (CSR)

    Create a CSR for the broker:

        keytool -alias kafka1 -keystore kafka.server.keystore.jks -certreq -file cert-file -storepass kafka-broker -keypass kafka-broker

    1.5. Sign the CSR with the CA

    Sign the broker's CSR using the CA certificate:

        openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:kafka-broker -extensions SAN -extfile <(printf "\n[SAN]\nsubjectAltName=DNS:kafka1,DNS:localhost")

    1.6. Import the CA Certificate into the Keystore

    Import the CA certificate into the broker's keystore:

        keytool -importcert -keystore kafka.server.keystore.jks -alias ca-root -file ca-cert -storepass kafka-broker -keypass kafka-broker -noprompt

    1.7. Import the Signed Certificate into the Keystore

    Finally, import the signed certificate into the broker's keystore:

        keytool -keystore kafka.server.keystore.jks -alias kafka1 -import -file cert-signed -storepass kafka-broker -keypass kafka-broker -noprompt


Step 2: Correct the server.properties Files

    Ensure that the user_bob password is correctly set in the server.properties files on kafka2 and kafka3.

    Incorrect (old) line:

        user_bob="b0b-secret"

    Correct (new) line:

        user_bob="bob-secret"


Step 3: Run Docker-Compose and Test Kafka Brokers

    After correcting the configuration and SSL setup, run the following commands to bring up the Kafka services and test the brokers:

        Start Kafka Cluster using Docker Compose:

            docker-compose up -d

        Client:

        To use a kafka client, exec into the kfkclient container which contains the Kafka CLI and other tools necessary for troubleshooting Kafka. THe kfkclient container also contains a properties file mounted to /opt/client, which can be used to define the client properties for communicating with Kafka.

            docker exec -it kfkclient bash

        Test Connection with Kafka Brokers: Use the following commands to list topics for each Kafka broker:

            Kafka1 (Port: 19092):

                kafka-topics --bootstrap-server kafka1:19092 --command-config /opt/client/client.properties --list

            Kafka2 (Port: 29092):

                kafka-topics --bootstrap-server kafka2:29092 --command-config /opt/client/client.properties --list

            Kafka3 (Port: 39092):

                kafka-topics --bootstrap-server kafka3:39092 --command-config /opt/client/client.properties --list


Step 4: Verify Successful Authentication:

    If everything is set up correctly, the topic list should appear without authentication errors, confirming that the SSL and password issues have been resolved.



Results:

![alt text](<images/Screenshot from 2024-09-24 15-44-49.png>)
![alt text](<images/Screenshot from 2024-09-24 15-45-10.png>)
![alt text](<images/Screenshot from 2024-09-24 15-46-30.png>)              
![alt text](<images/Screenshot from 2024-09-24 15-46-36.png>)
![alt text](<images/Screenshot from 2024-09-24 15-48-30.png>)
![alt text](<images/Screenshot from 2024-09-24 15-48-36.png>)


