pipeline {
  agent {
    node {
      label 'maven'
    }
  }
  
  environment {
    DOCKER_CREDENTIAL_ID = 'docker-registry-credential'
    KUBECONFIG_CREDENTIAL_ID = 'kubeconfig-credential'
    REGISTRY = 'docker.io'
    NAMESPACE = 'web-apps'
    APP_NAME = 'lamp-stack'
  }
  
  stages {
    stage('拉取代碼') {
      steps {
        git(url: 'https://github.com/yung598/lamp-project.git', credentialsId: 'github-credential', branch: 'main')
      }
    }
    
    stage('創建命名空間') {
      steps {
        container('maven') {
          withCredentials([kubeconfigFile(credentialsId: env.KUBECONFIG_CREDENTIAL_ID, variable: 'KUBECONFIG')]) {
            sh 'kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -'
          }
        }
      }
    }
    
    stage('創建 MySQL 密鑰') {
      steps {
        container('maven') {
          withCredentials([kubeconfigFile(credentialsId: env.KUBECONFIG_CREDENTIAL_ID, variable: 'KUBECONFIG')]) {
            sh '''
              kubectl create secret generic mysql-secret \
                --from-literal=password=your-secure-password \
                -n ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
            '''
          }
        }
      }
    }
    
    stage('部署應用') {
      steps {
        container('maven') {
          withCredentials([kubeconfigFile(credentialsId: env.KUBECONFIG_CREDENTIAL_ID, variable: 'KUBECONFIG')]) {
            sh '''
              kubectl apply -f k8s/persistent-volume.yaml -n ${NAMESPACE}
              kubectl apply -f k8s/mysql.yaml -n ${NAMESPACE}
              kubectl apply -f k8s/php73.yaml -n ${NAMESPACE}
              kubectl apply -f k8s/php80.yaml -n ${NAMESPACE}
              kubectl apply -f k8s/nginx.yaml -n ${NAMESPACE}
            '''
          }
        }
      }
    }
    
    stage('驗證部署') {
      steps {
        container('maven') {
          withCredentials([kubeconfigFile(credentialsId: env.KUBECONFIG_CREDENTIAL_ID, variable: 'KUBECONFIG')]) {
            sh '''
              # 等待所有 Pod 就緒
              kubectl wait --for=condition=ready pod -l app=mysql -n ${NAMESPACE} --timeout=300s
              kubectl wait --for=condition=ready pod -l app=php73 -n ${NAMESPACE} --timeout=300s
              kubectl wait --for=condition=ready pod -l app=php80 -n ${NAMESPACE} --timeout=300s
              kubectl wait --for=condition=ready pod -l app=nginx -n ${NAMESPACE} --timeout=300s
              
              # 創建測試 PHP 文件
              kubectl exec -n ${NAMESPACE} $(kubectl get pods -n ${NAMESPACE} -l app=nginx -o name | head -n 1) -- \
                /bin/bash -c 'echo "<?php phpinfo(); ?>" > /usr/share/nginx/html/info.php'
                
              # 獲取服務 URL
              echo "應用已部署完成！"
              echo "訪問地址: http://$(kubectl get svc nginx -n ${NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].ip}')/info.php"
            '''
          }
        }
      }
    }
  }
}
