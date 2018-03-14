# Initialize the Cluster<a name="initialize-cluster"></a>

Complete the steps in the following topics to initialize your AWS CloudHSM cluster\.

**Note**  
Before you initialize the cluster, review the process by which you can [verify the identity and authenticity of the HSMs](verify-hsm-identity.md)\. This process is optional and works only until a cluster is initialized\. After the cluster is initialized, you cannot use this process to get your certificates or verify the HSMs\. 


+ [Get the Cluster CSR](#get-csr)
+ [Sign the CSR](#sign-csr)
+ [Initialize the Cluster](#initialize)

## Get the Cluster CSR<a name="get-csr"></a>

Before you can initialize the cluster, you must and sign a certificate signing request \(CSR\) that is generated by the cluster's first HSM\. If you followed the steps to [verify the identity of your cluster's HSM](verify-hsm-identity.md), you already have the CSR and you can sign it\. Otherwise, retrieve the CSR now by using the [AWS CloudHSM console](https://console.aws.amazon.com/cloudhsm/), the [AWS Command Line Interface \(AWS CLI\)](https://aws.amazon.com/cli/), or the AWS CloudHSM API\. 

**Retrieve the CSR \(console\)**

1. Open the AWS CloudHSM console at [https://console\.aws\.amazon\.com/cloudhsm/](https://console.aws.amazon.com/cloudhsm/)\.

1. Choose **Initialize** next to the cluster that you [created previously](create-cluster.md)\. 

1. When the CSR is ready, you see a link to download it\.  
![\[Download certificate signing request page in the AWS CloudHSM console.\]](http://docs.aws.amazon.com/cloudhsm/latest/userguide/images/console-download-certificates.png)

   Choose **Cluster CSR** to download and save the CSR\.

**Retrieve the CSR \([AWS CLI](http://docs.aws.amazon.com/cli/latest/userguide/)\)**

+ At a command prompt, run the following [describe\-clusters](http://docs.aws.amazon.com/cli/latest/reference/cloudhsmv2/describe-clusters.html) command, which extracts the CSR and saves it to a file\. Replace *<cluster ID>* with the ID of the cluster that you [created previously](create-cluster.md)\. 

  ```
  $ aws cloudhsmv2 describe-clusters --filters clusterIds=<cluster ID> \
                                     --output text \
                                     --query 'Clusters[].Certificates.ClusterCsr' \
                                     > <cluster ID>_ClusterCsr.csr
  ```

**Retrieve the CSR \(AWS CloudHSM API\)**

+ Send a [http://docs.aws.amazon.com/cloudhsm/latest/APIReference/API_DescribeClusters.html](http://docs.aws.amazon.com/cloudhsm/latest/APIReference/API_DescribeClusters.html) request, and then extract and save the CSR from the response\.

## Sign the CSR<a name="sign-csr"></a>

Currently, you must create a self\-signed signing certificate and use it to sign the CSR for your cluster\. One way to do this is by using a Linux shell and OpenSSL\. You do not need the AWS CLI for this step, and the shell does not need to be associated with your AWS account\. To sign the CSR, you must do the following:

+ Retrieve the CSR \(see [Get the Cluster CSR](#get-csr)\)\.

+ Create a private key\.

+ Use the private key to create a signing certificate\.

+ Use the signing certificate to sign your cluster CSR\.

**To sign the CSR \(OpenSSL\)**

1. Use the following command to create a private key\. We recommend that you save this key in a safe place\. Any client that tries to communicate with an HSM requires this key\. 

   ```
   $ openssl genrsa -aes256 -out customerCA.key 2048
   Generating RSA private key, 2048 bit long modulus
   ........+++
   ............+++
   e is 65537 (0x10001)
   Enter pass phrase for customerCA.key:
   Verifying - Enter pass phrase for customerCA.key:
   ```

1. Use the following command and the private key that you created in the previous step to create a signing certificate\. The certificate is valid for 10 years \(3652 days\)\. Read the on\-screen instructions and follow the prompts\. 

   ```
   $ openssl req -new -x509 -days 3652 -key customerCA.key -out customerCA.crt
   Enter pass phrase for customerCA.key:
   You are about to be asked to enter information that will be incorporated
   into your certificate request.
   What you are about to enter is what is called a Distinguished Name or a DN.
   There are quite a few fields but you can leave some blank
   For some fields there will be a default value,
   If you enter '.', the field will be left blank.
   -----
   Country Name (2 letter code) [AU]:
   State or Province Name (full name) [Some-State]:
   Locality Name (eg, city) []:
   Organization Name (eg, company) [Internet Widgits Pty Ltd]:
   Organizational Unit Name (eg, section) []:
   Common Name (e.g. server FQDN or YOUR name) []:
   Email Address []:
   ```

   This command creates a file named `customerCA.crt`\. Use this file to initialize the cluster\.

1. Sign the cluster's CSR with the signing certificate that you created in the previous step\. Replace *<cluster ID>* with the ID of the cluster that you created previously\. 

   ```
   $ openssl x509 -req -days 3652 -in <cluster ID>_ClusterCsr.csr \
                                 -CA customerCA.crt \
                                 -CAkey customerCA.key \
                                 -CAcreateserial \
                                 -out <cluster ID>_CustomerHsmCertificate.crt
   Signature ok
   subject=/C=US/ST=CA/O=Cavium/OU=N3FIPS/L=SanJose/CN=HSM:<HSM identifer>:PARTN:<partition number>, for FIPS mode
   Getting CA Private Key
   Enter pass phrase for customerCA.key:
   ```

   This command creates a file named `<cluster ID>_CustomerHsmCertificate.crt`\. Use this file as the signed certificate when you initialize the cluster\. 

## Initialize the Cluster<a name="initialize"></a>

Use your signed HSM certificate and your signing certificate to initialize your cluster\. You can use the [AWS CloudHSM console](https://console.aws.amazon.com/cloudhsm/), the [AWS CLI](https://aws.amazon.com/cli/), or the AWS CloudHSM API\. 

**To initialize a cluster \(console\)**

1. Open the AWS CloudHSM console at [https://console\.aws\.amazon\.com/cloudhsm/](https://console.aws.amazon.com/cloudhsm/)\.

1. Choose **Initialize** next to the cluster that you created previously\.

1. On the **Download certificate signing request** page, choose **Next**\. If **Next** is not available, first choose one of the CSR or certificate links\. Then choose **Next**\.

1. On the **Sign certificate signing request \(CSR\)** page, choose **Next**\.

1. On the **Upload the certificates** page, do the following:

   1. Next to **Cluster certificate**, choose **Upload file**\. Then select the HSM certificate that you signed previously\. If you completed the steps in the previous section, select the file named `<cluster ID>_CustomerHsmCertificate.crt`\.

   1. Next to **Issuing certificate**, choose **Upload file**\. Then select your signing certificate\. If you completed the steps in the previous section, select the file named `customerCA.crt`\. 

   1. Choose **Upload and initialize**\.

**To initialize a cluster \([AWS CLI](http://docs.aws.amazon.com/cli/latest/userguide/)\)**

+ At a command prompt, run the [initialize\-cluster](http://docs.aws.amazon.com/cli/latest/reference/cloudhsmv2/initialize-cluster.html) command\. Provide the following: 

  + The ID of the cluster that you created previously\.

  + The HSM certificate that you signed previously\. If you completed the steps in the previous section, it's saved in a file named `<cluster ID>_CustomerHsmCertificate.crt`\. 

  + Your signing certificate\. If you completed the steps in the previous section, the signing certificate is saved in a file named `customerCA.crt`\.

  ```
  $ aws cloudhsmv2 initialize-cluster --cluster-id <cluster ID> \
                                      --signed-cert file://<cluster ID>_CustomerHsmCertificate.crt \
                                      --trust-anchor file://customerCA.crt
  {
      "State": "INITIALIZE_IN_PROGRESS",
      "StateMessage": "Cluster is initializing. State will change to INITIALIZED upon completion."
  }
  ```

**To initialize a cluster \(AWS CloudHSM API\)**

+ Send an [http://docs.aws.amazon.com/cloudhsm/latest/APIReference/API_InitializeCluster.html](http://docs.aws.amazon.com/cloudhsm/latest/APIReference/API_InitializeCluster.html) request with the following:

  + The ID of the cluster that you created previously\.

  + The HSM certificate that you signed previously\.

  + Your signing certificate\.

After you initialize the cluster, proceed to [Launch a Client](launch-client-instance.md)\.