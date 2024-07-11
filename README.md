# Managing secrets on Kubernetes

Our client is currently utilizing Kubernetes on AWS (EKS + Terraform) and storing sensitive data, such as database passwords, in a configuration file that is saved alongside the code on Github. To ensure that the relevant secrets for the specific environment, such as staging or production, are loaded into the application pod, an ENV variable with the name of the environment is assigned.
We aim to assist our client in enhancing their approach to handling sensitive information.

## Solution Overview.

In Kubernetes, the fundamental principle is to securely store sensitive information, known as secrets, in external, highly secure, and fault-tolerant storage systems. This practice is crucial to safeguarding confidential data. To seamlessly integrate these secrets into Kubernetes applications, operators designed for this purpose operate within the cluster. These operators facilitate the synchronization of specified secrets from the external storage into Kubernetes' internal secrets vault.

Once securely stored within Kubernetes, these secrets can be conveniently injected into application pods. This injection can take various forms, including setting them as environment variables or mounting secret files directly into the pod's filesystem. This ensures that sensitive data, such as passwords, tokens, or encryption keys, remains protected and accessible only to authorized applications within the Kubernetes ecosystem.

![Secret diagram](images/secrets-diagram.jpeg)

## Requirements

**Data Security**: The solution must ensure the security of sensitive data such as passwords, tokens, and keys. It should encrypt secrets both at rest and in transit to protect against unauthorized access and data breaches[1].

**Access Control**: Implement fine-grained access control mechanisms to restrict who can access and modify secrets. Role-Based Access Control (RBAC) should be supported to define permissions.

**Audit Trail**: Enable auditing capabilities to track changes and access to secrets. This helps in compliance and troubleshooting efforts.

**Ease of Integration**: The solution should seamlessly integrate with Kubernetes clusters and existing tools commonly used in the Kubernetes ecosystem.

**Secure Storage Backend**: Utilize secure storage backends for storing secrets, which may include cloud-based solutions like AWS KMS or HashiCorp Vault.

**Secret Rotation**: Support for secret rotation is essential to regularly update credentials and reduce the attack surface.

These requirements are crucial for selecting a secure and effective secret management and storage solution in a Kubernetes environment.

## Overview of available tools

### Secret store

Here are three options to improve the way your client saves and manages secrets.

#### AWS Secrets Manager:

**Optimal for**: AWS-native applications, ease of integration with other AWS services.
**Why**: Secrets Manager is a managed service specifically designed for storing and managing secrets, such as database passwords and API keys. It provides rotation and auditing capabilities, making it suitable for applications running on AWS that require seamless integration with AWS services. Secrets Manager simplifies secret management tasks, but it's primarily focused on AWS-native applications and may have limitations for non-AWS resources.
Source: [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/ "AWS Secrets Manager")

#### AWS Systems Manager Parameter Store:

**Optimal for**: Storing configuration data and lightweight secret management within AWS.
**Why**: Parameter Store is a part of AWS Systems Manager and is primarily designed for storing configuration parameters, but it can also be used to store lightweight secrets. It's suitable when you need to centralize configuration data for various AWS resources. While it can handle secrets, it lacks some of the advanced secret management features found in dedicated solutions like Secrets Manager and Vault.
Source: [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)

#### HashiCorp Vault:

**Optimal for**: Comprehensive secret management across multi-cloud or hybrid environments.
**Why**: Vault is a highly versatile and secure secret management tool that works across different cloud providers and on-premises environments. It offers advanced features like dynamic secrets, fine-grained access control, and support for multiple authentication methods. Vault is an excellent choice when you have a diverse set of resources, including those outside of AWS, and require a centralized, robust secret management solution.
Source: [HashiCorp Vault](https://www.vaultproject.io/ "HashiCorp Vault")


All options provide secure and scalable ways to manage secrets, and they can integrate well with Kubernetes on AWS while reducing the risk associated with storing sensitive data in code repositories. Your client can choose the one that aligns best with their requirements and capacity.

### Secrets managment in Kubernetes

Here are two options to help your client improve the way they save and manage secrets in their Kubernetes on AWS (EKS) environment while considering their small team and limited capacity for self-hosted solutions.

#### Kubernetes External Secrets

**Implementation**: Implement Kubernetes External Secrets to centralize secret management while keeping secrets in external stores.

**Workflow**:
* Store secrets securely in an external secret store (e.g., AWS Secrets Manager or HashiCorp Vault).
* Configure Kubernetes External Secrets to fetch secrets from the external store and inject them as Kubernetes Secrets into the application pods.
* Maintain environment-specific configurations in Kubernetes ConfigMaps.
* Use a simple CI/CD pipeline to deploy ConfigMaps and application code.

**Advantages:**
* Centralizes secret management in external stores.
* Separates secrets from code repositories.
* Supports various external secret management solutions.
* Requires minimal changes to the existing workflow.

Source: [Kubernetes External Secrets](https://github.com/external-secrets/external-secrets "Kubernetes External Secrets")

#### Kubernetes Secrets Store CSI Driver:

**Implementation:** Utilize the Kubernetes Secrets Store CSI Driver to enable secret retrieval from external secret stores without modifying application code.

**Workflow:**:
* Store secrets securely in an external secret store.
* Deploy the Kubernetes Secrets Store CSI Driver in the cluster.
* Configure the CSI Driver to fetch secrets from the external store and present them as volumes in the pods.
* Update the application deployment to mount secret volumes as files or environment variables.

**Advantages:**

* Keeps secrets separate from application code.
* Provides dynamic, on-demand secret retrieval.
* Supports various external secret stores.
* Requires minimal code changes.

Source: [Kubernetes Secrets Store CSI Driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver "Kubernetes Secrets Store CSI Driver")

Each of these options offers improved secret management while considering your client's small team and limited capacity for self-hosted solutions. The choice depends on their preferences and willingness to adopt external secret management solutions or implement GitOps practices.


## Considerations:

Ultimately, the optimal option depends on your specific requirements and the level of integration you need with external secret management systems. Consider your security, compliance, and operational needs when making your decision. It may also be beneficial to consult with your team or a Kubernetes expert to determine the best fit for your use case.

If you primarily use AWS services and value simplicity and automation, AWS Secrets Manager may be the better choice. Using HashiCorp Vault results in extra expenses for maintaining and managing infrastructure.

When deciding between Kubernetes External Secrets and Kubernetes Secrets Store CSI, both operators offer similar functionality. Ultimately, the choice comes down to personal preference and the specific features needed. However, in their documentation, Amazon suggests utilizing the AWS Secrets Manager + Kubernetes Secrets Store CSI Driver bundle. Therefore, we would be to recommend using this particular bundle.

## Implementation

Learn about the bundle of AWS Secrets Manager and Kubernetes Secrets Store CSI Driver.

##### Step 1.
First of all, we need to install two components in our cluster.
* [Kubernetes Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/ "Kubernetes Secrets Store CSI Driver")
* [Config Provider for Secret Store CSI Driver](https://github.com/aws/secrets-store-csi-driver-provider-aws/tree/main "Config Provider for Secret Store CSI Driver")

The installation process is briefly described [here](https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html "here"). We will not describe it in detail.

##### Step 2.
Now create a test secret:

    aws --region "$REGION" secretsmanager  create-secret --name MySecret --secret-string '{"username":"memeuser", "password":"hunter2"}'

Now create the SecretProviderClass which tells the AWS provider which secrets are to be mounted in the pod.

    apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
    kind: SecretProviderClass
    metadata:
      name: nginx-deployment-aws-secrets
    spec:
      provider: aws
      parameters:
        objects: |
            - objectName: "MySecret"
              objectType: "secretsmanager"


Finally we can deploy our pod.
In the Kubernetes manifest, we define a volume and specify the secret provider class. Finally, we mount the volume in the pod.

          ............
		  volumes:
          - name: secrets-store-inline
            csi:
              driver: secrets-store.csi.k8s.io
              readOnly: true
              volumeAttributes:
                secretProviderClass: "nginx-deployment-aws-secrets"
          ...............
            volumeMounts:
            - name: secrets-store-inline
              mountPath: "/mnt/secrets-store"
              readOnly: true

To verify the secret has been mounted properly, See the example below:

    kubectl exec -it $(kubectl get pods | awk '/nginx-deployment/{print $1}' | head -1) cat /mnt/secrets-store/MySecret; echo


This is a small example of how to use the presented bundle. We can even translate secrets into Kubernetes POD environment variables.

test2-312
