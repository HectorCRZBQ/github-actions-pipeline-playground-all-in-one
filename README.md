# GitHub Actions Pipeline Playground

Este repositorio contiene una implementación de referencia de un pipeline de CI/CD utilizando GitHub Actions, Terraform y AWS. El pipeline despliega una aplicación web estática en AWS S3 y gestiona los artefactos de build.

## Requisitos Previos

- Cuenta de AWS
- AWS CLI configurado localmente
- Terraform instalado localmente
- Node.js y npm instalados
- Git

## Configuración Inicial

1. **Fork del Repositorio**
   ```bash
   # Crear un fork de este repositorio en tu cuenta de GitHub
   # Clonar tu fork localmente
   git clone https://github.com/TU-USUARIO/github-actions-pipeline-playground-all-in-one.git
   cd github-actions-pipeline-playground-all-in-one
   ```

2. **Crear Usuario IAM para GitHub Actions**


![Imagen](/images/image-1.png)
   
   Crear un nuevo usuario IAM en AWS con los siguientes permisos:

   ```json
   {
      "Version": "2012-10-17",
      "Statement": [
         {
            "Effect": "Allow",
            "Action": [
               "s3:CreateBucket",
               "s3:DeleteBucket",
               "s3:ListBucket",
               "s3:Get*",
               "s3:*Object",
               "s3:PutBucketPolicy",
               "s3:PutBucketPublicAccessBlock",
               "s3:PutBucketVersioning",
               "s3:PutBucketWebsite",
               "s3:GetBucketCORS",
               "s3:PutBucketCORS"
            ],
            "Resource": [
               "arn:aws:s3:::github-actions-pipeline-web-*",
               "arn:aws:s3:::github-actions-pipeline-web-*/*",
               "arn:aws:s3:::github-actions-pipeline-artifacts-*",
               "arn:aws:s3:::github-actions-pipeline-artifacts-*/*",
               "arn:aws:s3:::terraform-state-*",
               "arn:aws:s3:::terraform-state-*/*"
            ]
         },
         {
            "Effect": "Allow",
            "Action": [
               "dynamodb:CreateTable",
               "dynamodb:DeleteTable",
               "dynamodb:PutItem",
               "dynamodb:GetItem",
               "dynamodb:DeleteItem",
               "dynamodb:UpdateItem"
            ],
            "Resource": "arn:aws:dynamodb:*:*:table/terraform-lock"
         }
      ]
   }
   ```

![Imagen](/images/image-2.png)

![Imagen](/images/image-3.png)

Incorporamos la nueva política que hemos definido

![Imagen](/images/image-4.png)

![Imagen](/images/image-5.png)

![Imagen](/images/image-6.png)

Le creamos los credenciales que le restan, le falta la Acces Key correspondiente donde se nos da el **AWS_ACCESS_KEY_ID** y el **AWS_SECRET_ACCESS_KEY**

![Imagen](/images/image-7.png)

Descargamos el csv para no perder los credenciales.

Defino las credenciales dentro de **aws configure**
- AWS Access Key ID [None]: En el CSV
- AWS Secret Access Key [None]: En el CSV
- Default region name [None]: eu-west-1        
- Default output format [None]:


3. **Preparar Backend de Terraform**
   ```bash
   
   # Activa unas credenciales de AWS:
   
   ## opción 1 (con aws configure ...)
   export AWS_DEFAULT_PROFILE=nombre_profile
   
   ## opción 2 (con variables de entorno)
   export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
   export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
   export AWS_DEFAULT_REGION=eu-west-1
   
   # Ejecutar script de setup que creará el bucket S3 y tabla DynamoDB para el backend   
   ./scripts/setup.sh
   
   # Tomar nota del nombre del bucket creado para el estado de Terraform
   ```

Modificamos los valores del Bucket con lo obtenido de la ejecucción del comando anterior **./scripts/setup.sh**:

![Imagen](/images/image-8.png)

 - En main.tf:

 Los dos campos con posibles modificacione son el **bucket** que recibe el nombre obtenido y la **region** si se ha decidido mosntarlo en una región distint a la predefinida.

![Imagen](/images/image-9.png)

 - En variables.tf:

![Imagen](/images/image-10.png)



4. **Primer Despliegue Manual**
   ```bash
   # Configurar el backend de Terraform e inicializar el backend remoto
   cd iac
   terraform init
   terraform plan
   terraform apply
   ```
![Imagen](/images/image-11.png)

Si da error prueba a ejecutar los siguientes comandos para intentar solucionarlo:

   ```bash
 - aws sts get-caller-identity  # Verifica que es el usuario que hemos declarado anteriormente
 - aws s3 ls   # Revisa que exista o no exista ese bucket
 - aws configure get region   # Verifica que la región coincide con la region definida previamente
   ```
 - Volvemos a ejecutar el comando **./scripts/setup.sh**
 - Incluimos los valores obtenidos dentro del archivo *main.tf* actualizando su contenido.
 - Accedemos a la carpeta **/iac** y volvemos a ejecutar el comando **terraform init**



![Imagen](/images/image-12.png)

![Imagen](/images/image-13.png)


5. **Configurar Secretos en GitHub**

   En tu repositorio de GitHub, navega a Settings > Secrets and variables > Actions y añade estos tres secretos de repositorio:
   - `AWS_ACCESS_KEY_ID`: Tu Access Key del usuario IAM creado
   - `AWS_SECRET_ACCESS_KEY`: Tu Secret Access Key del usuario IAM creado
   - `AWS_REGION`: La región de AWS donde desplegarás (ej: eu-west-1) 

![Imagen](/images/image-14.png)

## Uso del Pipeline

El pipeline se activará automáticamente con cada push a la rama main. También puedes:

Modificamos el conatenido del index.html de /src:

![Imagen](/images/image-15.png)

1. Realizar cambios en el código fuente (src/):
   ```bash
   # Editar src/index.html
   git add src/index.html
   git commit -m "Update website content"
   git push origin main
   ```


2. Verificar la ejecución del pipeline en la pestaña "Actions" de tu repositorio.

3. Visitar la URL del bucket S3 (disponible en los outputs de Terraform) para ver tu sitio desplegado.

## Estructura del Proyecto

- `.github/workflows/`: Configuración del pipeline de GitHub Actions
- `src/`: Código fuente de la aplicación web
- `iac/`: Código de Terraform para infraestructura
- `scripts/`: Scripts de utilidad



## Limpieza

Para evitar costos innecesarios, recuerda eliminar los recursos cuando ya no los necesites:

```bash
cd iac
terraform destroy
```

También deberás eliminar manualmente:
- El bucket de estado de Terraform
- La tabla DynamoDB de bloqueo

## FAQ

Puede existir un bloqueo de a nivel de cuenta para que los bucket de S3 no estén públicos, en este caso se desactiva temporalmente y se vuelve a activar una vez finalizada.
