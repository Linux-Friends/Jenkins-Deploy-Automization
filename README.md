# ☁️ Spring Boot Jar Auto-Deploy with Jenkins

## 1. ✨ 프로젝트 개요

- GitHub - Jenkins - Ubuntu 호스트 - Docker 기반 사이트에서
- Spring Boot 작업의 jar 블루일 후
- 다른 서버로 jar 전송 + 자동 실행(재실행) 기능 구축

---

## 2. 프로젝트 목표

### ▶ GitHub 에서 컴미팅 후 Jenkins가 jar 자료 생성

- CI 구성: Jenkins Pipeline 을 통해 jar build

### ▶ jar 바인드 마우널트 환경 통해 호스트에 jar 저장

- Docker Jenkins 커널에서 바깥 jar 파일 포지션을 복사

### ▶ 2번 서버(myserver02)에 jar 전송 + 실행

- scp 를 통해 jar 전송
- ssh 로 Spring Boot 실행 작업 (기존에 실행 중이면 kill 후 restart)

---

## 3. 초기 설계: `inotifywait + sh`

### 포지션

- GitHub 결함 없이 Jenkins 또는 호스트 자체가
- jar 파일 변경을 자체 감지 -> 외부 다음 작업 호출

### 스크립트

```bash
#!/bin/bash
WATCH_FILE="/home/ubuntu/jarappdir/app.jar"

inotifywait -m -e close_write --format "%w%f" "$WATCH_FILE" | while read FILE
do
    echo "\ud83d\udce6 JAR \ubcc0\uacbd \uac10\uc9c0\ub428: $FILE"
    scp -o StrictHostKeyChecking=no "$FILE" ubuntu@myserver02:/home/ubuntu/jarappdir/
done
```

### 평가

- 실행은 간단하고 빠르지만, Jenkins와 관리 분류
- Jenkins 레프리케이션, 블록 관리가 따로 보조되면 보성 단지

---

## 4. 설계 고도화: Jenkins Pipeline 전부 호환

### Jenkinsfile 예제

```groovy
pipeline {
  agent any
  environment {
    REMOTE_HOST = "myserver02"
    REMOTE_USER = "ubuntu"
    REMOTE_DIR = "/home/ubuntu/jarappdir"
    LOCAL_JAR = "/var/jenkins_home/jar-output/app.jar"
  }

  stages {
    stage('Build') {
      steps {
        sh './gradlew clean build --no-daemon'
      }
    }

    stage('Deploy') {
      steps {
        sh 'scp -o StrictHostKeyChecking=no $LOCAL_JAR $REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR'
        sh '''
          ssh -o StrictHostKeyChecking=no $REMOTE_USER@$REMOTE_HOST << EOF
            PID=$(pgrep -f app.jar)
            if [ ! -z "$PID" ]; then
              kill -9 $PID
              sleep 1
            fi
            nohup java -jar $REMOTE_DIR/app.jar > app.log 2>&1 &
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

### 평가

- Jenkins가 모두 호환 중심으로 작업
- Pipeline 에서 build, scp, ssh 가 일괄적으로 처리 되면
- 로그, 실행, 연락 관리가 더 쉽고 모든 행시 결과가 Jenkins UI에 전부 다 나오기 때문에 실무적으로 포함된다.

---

✅ **참고**

- Jenkins 에서 ssh/scp 실행하려면 서버2에 대한 SSH key 등록 보조가 필요함
- systemd 서비스 로 재시작을 구성하면 중복 호출, 서비스 현황 확인이 가능해지므로 여부 검토 값이 높음

