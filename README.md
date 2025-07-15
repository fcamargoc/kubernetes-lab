# kubernetes-lab
Este proyecto realiza el despliegue de una aplicación frontend en AWS, utilizando los servicios de ECR para el almacenamiento de imágenes Docker y EKS para la orquestación de contenedores. Adicionalmente, la infraestructura es monitorizada mediante ArgoCD para la gestión del ciclo de vida del despliegue. Finalmente, el objetivo principal del proyecto es asegurar y controlar el proceso de despliegue mediante análisis de seguridad con las herramientas Trivy y Semgrep.

# Step 1

## 1. Por medio del archivo Cluster.yaml definimos la configuracion del cluster, utilizaremos eksctl de aws para la creacion.


## 2. en consola ejecuta el siguiente comando:

eksctl create cluster -f cluster.yaml

esta creación puede tardar unos 10 minutos, revisa que tengas configurado en tu maquina la herramienta de eksctl junto con las credenciales de aws.

al finalizar verifica la creacion con el siguiente comando:

kubectl get nodes

# Step 2

## 1. instalación de Argo CD en el Cluster.

Argo sera el motor de GitOps

crea el siguiente Namespace para Argo con el siguiente comando:

kubectl create namespace argocd

## 2. Instala Argo CD con el siguiente comando:

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

## 3. Accede a la interfaz de usuario de Argo CD:
Por defecto, el servicio argocd-server es de tipo ClusterIP. Para acceder, puedes hacer un port-forward:

kubectl port-forward svc/argocd-server -n argocd 8080:443

abre tu navegador y digita la siguiente url  https://localhost:8080

# 4. Obten la contraseña inicial de Argo CD

en Powershell ejecuta el siguiente script:

[System.Text.Encoding]::UTF8.GetString([Convert]::FromBase64String(
(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}")
))

el usuario predeterminado es admin

# Step 3

## 1. Crea un repositorio ECR con el siguiente script

aws ecr create-repository --repository-name gitops-web-app --region us-east-1 # Ajusta la región

Copia el URI de tu repositorio ECR. Lo necesitarás en el workflow.

## 2. Configura los secreto de aws 
En tu repositorio de GitHub, ve a Settings > Secrets and variables > Actions > New repository secret.

AWS_ACCESS_KEY_ID: Tu clave de acceso de AWS.

AWS_SECRET_ACCESS_KEY: Tu clave secreta de AWS.

En tu repositorio de GitHub, ve a Settings > Secrets and variables > Actions > New repository variable.

AWS_REGION : la región

# Step 4

## 1. ajusta el manifiesto deployment.yaml

Reemplaza YOUR_ECR_REPOSITORY_URI con el URI de tu repositorio ECR

## 2. instala Nginx ingress controller con el siguiente script

helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace

Espera a que el External IP del servicio ingress-nginx-controller esté disponible:

kubectl get svc -n ingress-nginx

Copiarás esa IP o DNS para acceder a tu aplicación.

# Step 5

## 1. Configurar Argo CD Application
Ahora le diremos a Argo CD dónde encontrar tus manifiestos de Kubernetes.
En el archivo manifests/applications/web-app.yaml.  Reemplaza https://github.com/tu-usuario/kubernetes-gitops-lab.git con la URL de tu propio repositorio.

## 2.Desplegar la Aplicación con GitOps
Añade todos los archivos al repositorio y haz un commit inicial:

git add .
git commit -m "Initial setup for GitOps Lab"
git push origin main

Esto activará tu workflow de GitHub Actions, que construirá y escaneará la imagen.

## 3. Registra la aplicación en Argo CD:
Una vez que la imagen esté en ECR, aplica la definición de la aplicación a tu clúster:

kubectl apply -n argocd -f manifests/applications/web-app.yaml

## 4. Observa en Argo CD:
Vuelve a la UI de Argo CD (https://localhost:8080). Deberías ver una nueva aplicación llamada simple-web-app. Argo CD detectará los manifiestos en tu repositorio y comenzará a sincronizar el estado deseado con el clúster. Espera a que la aplicación esté en estado Healthy y Synced.

## 5. Accede a tu aplicación web:
Obtén la IP externa del servicio ingress-nginx-controller:

kubectl get svc -n ingress-nginx

Copia la EXTERNAL-IP o EXTERNAL-DNS y pégala en tu navegador. Deberías ver tu página web sencilla.

# Step 6

## 1. Comprobación y Verificación

GitHub Actions:

Verifica que el workflow CI/CD Pipeline se haya ejecutado correctamente.

Revisa los logs de los pasos de Trivy y Semgrep para ver los resultados de los escaneos de seguridad. Si Trivy encuentra vulnerabilidades críticas/altas, el paso debería fallar (según la configuración exit-code: '1').

Asegúrate de que las imágenes Docker se hayan subido a ECR.

Argo CD UI:

Confirma que la aplicación simple-web-app esté en estado Synced y Healthy.

Explora los recursos desplegados (Deployment, Service, Ingress).

Intenta hacer un cambio pequeño en app/index.html, haz git commit y git push. Observa cómo Argo CD detecta el cambio y sincroniza automáticamente el Deployment, actualizando tu página web.

