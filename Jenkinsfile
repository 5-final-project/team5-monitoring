pipeline {
  agent { label 'team5' }
  
  environment {
    COMPOSE_PROJECT_NAME = "team5-monitoring"
  }
  
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    
    stage('Validate Configuration') {
      steps {
        sh '''
          # Prometheus 설정 파일 검증
          docker run --rm -v $(pwd)/prometheus/prometheus.yml:/prometheus.yml:ro \
            prom/prometheus:latest \
            promtool check config /prometheus.yml
          
          echo "Prometheus configuration is valid"
        '''
      }
    }
    
    stage('Deploy Monitoring Stack') {
      steps {
        sh '''
          # 기존 스택 중지 (에러 무시)
          docker-compose down || true
          
          # Team5 네트워크 생성 (이미 있으면 무시)
          docker network create team5-net || true
          
          # 새 스택 시작
          docker-compose up -d
          
          # 서비스 시작 대기
          sleep 30
          
          # 헬스체크
          curl -f http://localhost:9290/-/healthy || exit 1
          curl -f http://localhost:3600/api/health || exit 1
          
          echo "Monitoring stack deployed successfully"
        '''
      }
    }
    
    stage('Verify Targets') {
      steps {
        sh '''
          # Prometheus 타겟 상태 확인
          echo "Checking Prometheus targets..."
          
          # 모든 타겟이 UP 상태인지 확인
          targets_up=$(curl -s http://localhost:9290/api/v1/targets | grep -o '"health":"up"' | wc -l)
          total_targets=$(curl -s http://localhost:9290/api/v1/targets | grep -o '"health":"' | wc -l)
          
          echo "Targets UP: $targets_up / $total_targets"
          
          if [ "$targets_up" -gt 0 ]; then
            echo "At least some targets are healthy"
          else
            echo "No targets are healthy - check configuration"
            exit 1
          fi
        '''
      }
    }
    
    stage('Backup Configuration') {
      steps {
        sh '''
          # 설정 백업 생성
          timestamp=$(date +%Y%m%d_%H%M%S)
          mkdir -p backups
          
          # 현재 설정들 백업
          cp prometheus/prometheus.yml backups/prometheus_${timestamp}.yml
          
          # Docker Compose 설정 백업
          cp docker-compose.yml backups/docker-compose_${timestamp}.yml
          
          echo "Configuration backed up with timestamp: ${timestamp}"
        '''
      }
    }
  }
  
  post {
    success {
      echo "Team5 monitoring stack deployed successfully at ${new Date()}"
      sh '''
        echo "Access URLs:"
        echo "- Prometheus: http://localhost:9290"
        echo "- Grafana: http://localhost:3600 (admin/team5admin)"
        echo "- Node Exporter: http://localhost:9105"
      '''
    }
    failure {
      echo "Monitoring deployment failed at ${new Date()}"
      sh '''
        echo "Collecting logs for debugging..."
        docker-compose logs --tail=50 || true
        docker ps -a | grep team5 || true
      '''
    }
    always {
      sh '''
        # 정리 작업
        docker system prune -f || true
      '''
    }
  }
}