# ☁️ Spring Boot Jar Auto-Deploy with Jenkins

## 1. ✨ 프로젝트 개요

- GitHub → Jenkins → Ubuntu 호스트(Docker 기반)에서 Spring Boot 앱의 jar 빌드 및 자동 배포
- 빌드된 jar 파일을 감지하여, 2번 서버(myserver02)에 전송하고 Spring Boot 앱을 자동 실행(또는 재시작)
- 초기에는 `inotifywait` 기반 스크립트를 사용했으며, 이후 Jenkins Pipeline으로 고도화함

---

## 2. 프로젝트 목표

### ▶ GitHub 커밋 시 Jenkins가 자동으로 jar 생성
- CI 구성: Jenkins Pipeline으로 jar 빌드 자동화

### ▶ 바인드 마운트를 통해 jar 파일을 호스트 디렉토리에 저장
- Docker 컨테이너(Jenkins)와 호스트 공유 디렉토리를 설정하여 빌드 결과물을 외부로 노출

### ▶ 2번 서버(myserver02)로 jar 파일 전송 및 실행
- `scp`로 대상 서버에 jar 전송
- `ssh`로 기존 프로세스 종료 후 jar 앱 재실행 처리

---

## 3. 초기 설계: `inotifywait + sh`

### 🔧 설계 개요
- Jenkins를 통하지 않고 호스트에서 직접 jar 파일 변경을 감지
- 변경 시 자동으로 `scp` 전송 및 원격 실행

### 📜 스크립트 예시

```bash
#!/bin/bash

WATCH_FILE="/home/ubuntu/jarappdir/app.jar"
REMOTE_USER="ubuntu"
REMOTE_HOST="192.168.0.112"
REMOTE_PORT=23
REMOTE_PATH="/home/ubuntu/deploy"
TARGET_FILE="$REMOTE_PATH/app.jar"

inotifywait -m -e close_write --format "%w%f" "$WATCH_FILE" | while read FILE
do
    echo "📦 JAR 변경 감지됨: $FILE → $REMOTE_USER@$REMOTE_HOST:$REMOTE_PATH"

    scp -o StrictHostKeyChecking=no -P "$REMOTE_PORT" "$FILE" "$REMOTE_USER@$REMOTE_HOST:$TARGET_FILE"

    echo "🔁 원격 재시작 시도"
    ssh -o StrictHostKeyChecking=no -p "$REMOTE_PORT"  $REMOTE_USER@$REMOTE_HOST << EOF
    PID=\$(pgrep -f $TARGET_FILE)
    if [ ! -z "\$PID" ]; then
      kill -9 \$PID
      sleep 1
    fi
    nohup java -jar $TARGET_FILE --server.port=8088 > app.log 2>&1 &
EOF

done
```

### 💬 평가
- 스크립트 기반이라 간단하고 빠르게 동작하지만, 프로세스와 로깅 관리가 분산됨
- Jenkins와 분리되기 때문에 CI/CD 흐름과 통합되지 않는 단점 있음

---

## 4. 설계 고도화: Jenkins Pipeline 기반 자동화

### Jenkinsfile 예시

```groovy
pipeline {
  agent any
  environment {
    REMOTE_HOST = "myserver02"
    REMOTE_USER = "ubuntu"
    REMOTE_PORT = "23"
    REMOTE_PATH = "/home/ubuntu/deploy"
    LOCAL_JAR = "/var/jenkins_home/jar-output/app.jar"
    TARGET_FILE = "$REMOTE_PATH/app.jar"
  }

  stages {
    stage('Build') {
      steps {
        sh './gradlew clean build --no-daemon'
      }
    }

    stage('Deploy') {
      steps {
        sh 'scp -o StrictHostKeyChecking=no -P $REMOTE_PORT $LOCAL_JAR $REMOTE_USER@$REMOTE_HOST:$TARGET_FILE'
        sh '''
          ssh -o StrictHostKeyChecking=no -p $REMOTE_PORT $REMOTE_USER@$REMOTE_HOST << EOF
            PID=$(pgrep -f $TARGET_FILE)
            if [ ! -z "$PID" ]; then
              kill -9 $PID
              sleep 1
            fi
            nohup java -jar $TARGET_FILE --server.port=8088 > app.log 2>&1 &
          EOF
        '''
      }
    }
  }

  post {
    success { echo '✅ 배포 성공' }
    failure { echo '❌ 실패: 로그 확인 필요' }
  }
}
```

### 💬 평가
- Jenkins에서 전체 프로세스를 제어함으로써 통합 관리가 용이
- 빌드, 배포, 로그 출력이 한 곳(Jenkins)에서 이루어짐
- 실무적인 구조에 적합하며, 에러 핸들링 및 알림 확장도 용이함

---

✅ **참고 사항**

- Jenkins에서 ssh/scp 사용 시 대상 서버에 공개키 등록 필요
- 포트 번호가 22번이 아닌 경우 `-P <포트>` 옵션 추가 필수
- 대상 서버에서 Spring Boot 앱을 서비스로 등록(systemd)하면 더욱 안정적인 재시작 가능

