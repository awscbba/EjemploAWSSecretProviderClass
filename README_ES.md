# Ejemplo de "Secrets Service Class"

## Descripción

Este es un ejemplo de cómo otorgar acceso de los secretos guardados en AWS Secrets manager a los pods de un cluster EKS.

## Ambiente de desarrollo

Para facilitar el desarrollo hemos usado [Devbox](https://www.jetify.com/devbox/docs/).

#### Configurar Devbox

Para instalar Devbox:

```sh
curl -fsSL https://get.jetify.com/devbox | bash
```

Iniciar una shell virtual:

```sh
devbox shell
```

# Paso 1: Configurar el control de acceso

## Crear el secreto

Necesitamos tener un secreto creado inicialmente:

```sh
aws secretsmanager create-secret \
    --name TestSecret \
    --description "My test secret created with the CLI." \
    --secret-string "{\"user\":\"sergioi\",\"password\":\"XXXyyyZZZ000\"}"
``` 
Tomar nota de la dirección ARN del secreto creado. En nuestro caso es: `arn:aws:secretsmanager:us-east-1:142728997126:secret:TestSecret-ZQxKpG` .

# 1. Crear un proveedor OIDC en IAM para el cluster

```sh
cluster_name=basic-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo $oidc_id
```

Determinar si un proveedor OIDC con tu ID de cluster generado existe actualmente en el IAM de tu cuenta.

```sh
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

Si retorno salida, ya tienes un proveedor OIDC en IAM para tu cluster, entonces puedes saltar este paso al siguiente. Si no hubo salida entonces debes crear un proveedor OIDC en IAM para tu cluster.

Crear un proveedor OIDC en IAM para tu cluster con el siguiente comando.

```sh
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

## 2. Asignar un role en IAM a una cuenta de servicio de Kubernetes

Crear el archivo que contiene la política que permite acceder a los secretos de Secrets Manager a los pods:

```sh
cat >asm-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "BasePermissions",
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": "arn:aws:secretsmanager:us-east-1:142728997126:secret:TestSecret-ZQxKpG"
        }
    ]
}
EOF
```

Crear la política:

```sh
aws iam create-policy --policy-name my-asm-policy --policy-document file://asm-policy.json
```
Tomar nota del ARN de la política, y exportarla al entorno como variable.

```sh
export policy_arn=arn:aws:iam::111122223333:policy/my-asm-policy
```

Crear un rol en IAM y asociarlo con una cuenta de servicio de Kubernetes. Es posible usar eksctl o el AWS CLI.

```sh
eksctl create iamserviceaccount --name my-asm-service-account --namespace default --cluster $cluster_name --role-name my-asm-role --attach-policy-arn $policy_arn --approve
```

Confirmar que la política de confianza del rol en IAM está configurada correctamente.

```sh
aws iam get-role --role-name my-role --query Role.AssumeRolePolicyDocument
```

Confirmar que la política que agregaste a tu rol en el paso previo está agregada al rol.

```sh
aws iam list-attached-role-policies --role-name my-role --query AttachedPolicies[].PolicyArn --output text
```
Nota: No he podido hacer funcionar éste paso correctamente.

Ver la versión de la politica.

```sh
aws iam get-policy --policy-arn $policy_arn
```

Ver el contenido de la política para asegurarse que incluye todos los permisos que el Pod necesita. De ser necesario reemplaza 1 en el comando mostrado a continuación con la versión respectiva.

```sh
aws iam get-policy-version --policy-arn $policy_arn --version-id v1

```

Confirmar que la version de la cuenta de servicio de Kubernetes está anotada con el rol.

```sh
kubectl describe serviceaccount my-asm-service-account -n default
```

# Paso 2: Install y configurar ASCP

## Para instalar el ASCP usando Helm

1. Asegurarse que el repositio está apuntando a la última versión de los charts `helm repo update`.
2. Agregar el chart del Secrets Store CSI Driver.
```sh
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
```
1. Instalar el chart [2].
```sh
helm install -n kube-system csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --set syncSecret.enabled=true
```
1. Agregar el ASCP chart.
```sh
helm repo add aws-secrets-manager https://aws.github.io/secrets-store-csi-driver-provider-aws
```
1. Instalar chart. Para usar el endpoint FIPS, agregar la opción: `--set useFipsEndpoint=true`
```sh
helm install -n kube-system secrets-provider-aws aws-secrets-manager/secrets-store-csi-driver-provider-aws
```

# Paso 3: Identificar cuales secretos vamos a montar

En ésta guía hemos limitado el accesos sólo al secreto que hemos creado de inicio, pero es posible modificar la política para acceder a todos los secretos o a un grupo de ellos.

Para aprender más acerca de cómo montar los secretos en la definición del provider class, por favor visite el enlace en [3], en la seción de referencias.

# Paso 4: Montar los secretos como archivos y secretos en los Pods del cluster de Amazon EKS.

Para montar los secretos en Amazon EKS
- Aplicar el SecretProviderClass con el comando `kubectl apply -f SecretProviderClass.yaml`.
- Desplegar el Pod `kubectl apply -f my-deployment.yaml`.
- El ASCP mountará los archivos.

Deberían existir secretos creados luego de aplicar las definiciones:

```sh
kubectl -n default get secrets
```

Verificar el el montaje exíste:

```sh
cd /mnt/secrets-store
ls -hatlr
cat credentials
```

Verificar que las variables de entorno referencian a los secretos existentes:

```sh
env | grep ^DB*
```

## Referencias
1. https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html
2. https://stackoverflow.com/questions/73192808/env-variable-from-aws-secrets-manager-in-kubernetes
3. https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html#integrating_csi_driver_mount

## Créditos

- Creado por Sergio Rodríguez Inclán <srinclan@arcamo.org>
- AWS User Group Cochabamba: https://www.linkedin.com/groups/12914090/
