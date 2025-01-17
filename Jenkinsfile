pipeline {
   agent any

   environment {
       AWS_ACCOUNT_ID = '017820683847'
       AWS_DEFAULT_REGION = 'us-east-1'
       IMAGE_REPO_NAME = 'ecommerce-app'
       GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
       IMAGE_TAG = "${GIT_COMMIT_SHORT}-${BUILD_NUMBER}"
       REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
       NAMESPACE = 'ecommerce'
   }

   stages {
       stage('Build Backend Image') {
           steps {
               script {
                   sh """
                       docker build -t ecommerce-app-backend:${IMAGE_TAG} -f backend/Dockerfile.backend .
                       docker compose -f docker/docker-compose.yaml up -d

                       # Health check with improved error handling
                       HEALTH_CHECK_RETRIES=12
                       for i in \$(seq 1 \$HEALTH_CHECK_RETRIES); do
                           if curl -f http://localhost:5000/api/health; then
                               echo "Health check passed"
                               break
                           elif [ \$i -eq \$HEALTH_CHECK_RETRIES ]; then
                               echo "Health check failed after \$HEALTH_CHECK_RETRIES attempts"
                               exit 1
                           else
                               echo "Attempt \$i failed, retrying in 5 seconds..."
                               sleep 5
                           fi
                       done
                       
                       docker ps -a
                   """
               }
           }
       }

       stage('Push to ECR') {
           steps {
               withAWS(credentials: 'aws-access', region: env.AWS_DEFAULT_REGION) {
                   script {
                       sh """
                           aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${REPOSITORY_URI}
                           docker tag ecommerce-app-backend:${IMAGE_TAG} ${REPOSITORY_URI}:backend-${IMAGE_TAG}
                           docker push ${REPOSITORY_URI}:backend-${IMAGE_TAG}
                       """
                   }
               }
           }
       }

         stage('Configure Kubernetes') {
           steps {
               withAWS(credentials: 'aws-access', region: env.AWS_DEFAULT_REGION) {
                   sh '''
                       aws eks update-kubeconfig --name demo-eks-cluster --region $AWS_DEFAULT_REGION
                       kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
                   '''
                   
                   // Check and create storage class if needed
                   sh '''#!/bin/bash
                       if ! kubectl get storageclass ebs-sc &> /dev/null; then
                           cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  encrypted: "true"
EOF
                       else
                           echo "Storage class ebs-sc already exists"
                       fi
                   '''
               }
           }
       }

       stage('Deploy MySQL') {
           steps {
               withAWS(credentials: 'aws-access', region: env.AWS_DEFAULT_REGION) {
                   sh '''#!/bin/bash
                       # Clean up existing resources
                       kubectl delete pvc --namespace $NAMESPACE -l app=ecommerce-db --ignore-not-found=true
                       kubectl delete service,sts --namespace $NAMESPACE -l app=ecommerce-db --ignore-not-found=true
                       sleep 10
                       
                       # Deploy MySQL
                       helm upgrade --install ecommerce-mysql ./helm/ecommerce-app \
                           --namespace $NAMESPACE \
                           --set backend.enabled=false \
                           --set mysql.enabled=true \
                           --set mysql.storageClassName=ebs-sc \
                           --force \
                           --wait \
                           --timeout 5m

                       # Wait for MySQL to be ready
                       kubectl wait --namespace $NAMESPACE \
                           --for=condition=ready pod \
                           -l app=ecommerce-db \
                           --timeout=300s
                   '''
               }
           }
       }

       stage('Deploy Backend') {
           steps {
               withAWS(credentials: 'aws-access', region: env.AWS_DEFAULT_REGION) {
                   sh '''#!/bin/bash
                       # Clean up old backend
                       kubectl delete --namespace $NAMESPACE deployment,service -l app=ecommerce-backend --ignore-not-found=true
                       sleep 10
                       
                       # Deploy new backend
                       helm upgrade --install ecommerce-backend ./helm/ecommerce-app \
                           --namespace $NAMESPACE \
                           --set mysql.enabled=false \
                           --set backend.enabled=true \
                           --set backend.image.repository=${REPOSITORY_URI} \
                           --set backend.image.tag=backend-${IMAGE_TAG} \
                           --set backend.env.FLASK_APP=wsgi:app \
                           --set backend.env.FRONTEND_PATH=/app/frontend \
                           --force \
                           --wait \
                           --timeout 5m

                       # Wait for backend to be ready
                       kubectl wait --namespace $NAMESPACE \
                           --for=condition=ready pod \
                           -l app=ecommerce-backend \
                           --timeout=300s
                   '''
               }
           }
       }

       stage('Verify Deployment') {
           steps {
               withAWS(credentials: 'aws-access', region: env.AWS_DEFAULT_REGION) {
                   sh '''#!/bin/bash
                       echo "=== Final Deployment Status ==="
                       kubectl get pods,svc,deploy,sts -n $NAMESPACE
                       echo
                       echo "Application URL:"
                       kubectl get svc -n $NAMESPACE ecommerce-backend -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
                       echo
                   '''
               }
           }
       }
   }

   post {
       always {
           sh '''
               docker compose -f docker/docker-compose.yaml down || true
               docker rmi ${REPOSITORY_URI}:backend-${IMAGE_TAG} || true
               docker rmi ecommerce-app-backend:${IMAGE_TAG} || true
           '''
       }
       success {
           echo "Deployment successful!"
       }
       failure {
           withAWS(credentials: 'aws-access', region: env.AWS_DEFAULT_REGION) {
               sh '''#!/bin/bash
                   echo "=== Deployment Debug Info ==="
                   kubectl get pods,svc,deploy,sts -n $NAMESPACE
                   echo "=== Storage Classes ==="
                   kubectl get storageclass
                   echo "=== PVC Status ==="
                   kubectl get pvc -n $NAMESPACE
                   echo "=== MySQL Pod Description ==="
                   kubectl describe pod -n $NAMESPACE -l app=ecommerce-db || true
                   echo "=== MySQL Logs ==="
                   kubectl logs -n $NAMESPACE -l app=ecommerce-db --tail=100 || true
                   echo "=== Backend Pod Description ==="
                   kubectl describe pod -n $NAMESPACE -l app=ecommerce-backend || true
                   echo "=== Backend Logs ==="
                   kubectl logs -n $NAMESPACE -l app=ecommerce-backend --tail=100 || true
               '''
           }
           echo 'Deployment failed!'
       }
   }
}
