## AWS IoT JITP (One Time Setup)


### 1. Create IoT Policy
Save the following as any_type_thing_policy.json:
```bash
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iot:Connect"
      ],
      "Resource": "arn:aws:iot:*:*:client/${iot:Connection.Thing.ThingName}"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iot:Publish",
        "iot:Subscribe"
      ],
      "Resource": [
        "arn:aws:iot:*:*:topic/dt/${iot:Connection.Thing.ThingName}/*/telemetry",
        "arn:aws:iot:*:*:topic/dt/${iot:Connection.Thing.ThingName}/*/status",
        "arn:aws:iot:*:*:topic/dt/${iot:Connection.Thing.ThingName}/*/action",
        "arn:aws:iot:*:*:topic/$aws/things/${iot:Connection.Thing.ThingName}/shadow/update",
        "arn:aws:iot:*:*:topic/$aws/things/${iot:Connection.Thing.ThingName}/shadow/get",
        "arn:aws:iot:*:*:topic/$aws/things/${iot:Connection.Thing.ThingName}/shadow/delete"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "iot:Receive"
      ],
      "Resource": [
        "arn:aws:iot:*:*:topicfilter/dt/${iot:Connection.Thing.ThingName}/*/telemetry",
        "arn:aws:iot:*:*:topicfilter/dt/${iot:Connection.Thing.ThingName}/*/status",
        "arn:aws:iot:*:*:topicfilter/dt/${iot:Connection.Thing.ThingName}/*/action",
        "arn:aws:iot:*:*:topicfilter/$aws/things/${iot:Connection.Thing.ThingName}/shadow/update/accepted",
        "arn:aws:iot:*:*:topicfilter/$aws/things/${iot:Connection.Thing.ThingName}/shadow/get/accepted",
        "arn:aws:iot:*:*:topicfilter/$aws/things/${iot:Connection.Thing.ThingName}/shadow/update/delta"
      ]
    }
  ]
}

```

Create the policy:

``` bash
aws iot create-policy \
  --policy-name AnyTypeThing-policy \
  --policy-document file://any_type_thing_policy.json
```

### 2. Create IAM Role for Provisioning

Save this trust policy as aws_iot_trust_policy.json:

``` bash
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "iot.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

```

Create the role:

``` bash
aws iam create-role \
  --role-name iot-core-provisioning-role \
  --assume-role-policy-document file://aws_iot_trust_policy.json
```

### 3. Attach Registration Permissions to Role

``` bash
aws iam attach-role-policy \
  --role-name iot-core-provisioning-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSIoTThingsRegistration
```

### 4. Create the Provisioning Template

Save the following as jitp_provisioning_template.json:

``` bash
{
  "Parameters": {
    "AWS::IoT::Certificate::SerialNumber": {
      "Type": "String"
    }
  },
  "Resources": {
    "thing": {
      "Type": "AWS::IoT::Thing",
      "Properties": {
        "ThingName": {
          "Ref": "AWS::IoT::Certificate::SerialNumber"
        }
      }
    },
    "certificate": {
      "Type": "AWS::IoT::Certificate",
      "Properties": {
        "CertificateId": {
          "Ref": "AWS::IoT::Certificate::Id"
        },
        "Status": "ACTIVE"
      }
    },
    "policy": {
      "Type": "AWS::IoT::Policy",
      "Properties": {
        "PolicyName": "AnyTypeThing-policy"
      }
    }
  }
}

```

Create the provisioning template:

``` bash
aws iot create-provisioning-template \
  --template-name jitp-provisioning-template \
  --enabled \
  --type JITP \
  --provisioning-role-arn arn:aws:iam::<YOUR-ACCOUNT-ID>:role/iot-core-provisioning-role \
  --template-body file://jitp_provisioning_template.json

```

### 5. Generate Root CA Certificate

``` bash
openssl genrsa -out rootCA.key 2048
openssl req -new -sha256 -key rootCA.key -nodes -out rootCA.csr -config rootCA_openssl.conf
openssl x509 -req -days 3650 -extfile rootCA_openssl.conf -extensions v3_ca -in rootCA.csr -signkey rootCA.key -out rootCA.pem

```

### 6. Get Registration Code

``` bash
aws iot get-registration-code
```
### 7. Generate Verification Certificate

``` bash

openssl genrsa -out verificationCert.key 2048
openssl req -new -key verificationCert.key -out verificationCert.csr

```

### 8. Sign the Verification Certificate

``` bash
openssl x509 -req -in verificationCert.csr -CA rootCA.pem -CAkey rootCA.key \
  -CAcreateserial -out verificationCert.pem -days 500 -sha256
```

### 9. Register the CA Certificate

```bash
aws iot register-ca-certificate \
  --ca-certificate file://rootCA.pem \
  --verification-cert file://verificationCert.pem \
  --set-as-active \
  --allow-auto-registration \
  --registration-config templateName=jitp-provisioning-template
```

## AWS IoT JITP (Per-Device)

#### These steps are repeated for each new device that will be provisioned and connected to AWS IoT using your previously registered Root CA.


### 1. Generate Device Private Key

``` bash
openssl genrsa -out device.key 2048
```

### 2. Create Certificate Signing Request 

Replace <serial-number> with a unique identifier. 

```bash
openssl req -new -key device.key -out device.csr \
  -subj "/CN=device-<serial-number>"
```

### 3. Sign the Device Certificate with Root CA

``` bash
openssl x509 -req -in device.csr -CA rootCA.pem -CAkey rootCA.key \
  -CAserial rootCA.srl -CAcreateserial -out device.crt -days 365 -sha256
```

### 4. Prepare Files

 - device.crt
 - device.key
 - rootCA.pem
 - AmazonRootCA1.pem
  
### 5. Connect to MQTT 

A test to validate the steps

``` bash
mosquitto_pub \
  --cafile AmazonRootCA1.pem \
  --cert device.crt \
  --key device.key \
  -h <ATS-endpoint> \
  -p 8883 \
  -t "dt/device-<serial-number>/1/telemetry" \
  -m '{"status": "ok", "numbertest": 48.0}' \
  -d
```
