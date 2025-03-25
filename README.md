# ☁️ Spring Boot Jar Auto-Deploy with Jenkins

<br>

## 📍 Contributors

| <img src="https://github.com/DoomchitYJ.png" width="220" /> | <img src="https://github.com/imhaeunim.png" width="220" /> |
| --- | --- |
| 박영진<br/>[@DoomchitYJ](https://github.com/DoomchitYJ) | 임하은<br/>[@imhaeunim](https://github.com/imhaeunim) |

<br>

## 1. ✨ 프로젝트 개요

- GitHub → Jenkins에서 Spring Boot 앱의 jar 빌드
- 빌드된 jar 파일을 감지하여, 2번 서버(myserver02)에 전송하고 Spring Boot 앱을 자동 실행(또는 재시작)
- `inotifywait` 기반 스크립트를 사용했으며, 이후 Jenkins Pipeline으로 고도화 아이디어 제시

---

## 2. 프로젝트 흐름

### ▶ GitHub 커밋 시 Jenkins가 자동으로 jar 생성

- CI 구성: Jenkins Pipeline으로 jar 빌드 자동화

### ▶ 바인드 마운트를 통해 jar 파일을 호스트 디렉토리에 저장

- Docker 컨테이너(Jenkins)와 호스트 공유 디렉토리를 설정하여 빌드 결과물을 저장

### ▶ 2번 서버(myserver02)로 jar 파일 전송 및 실행

- inotify로 jar의 변경 감지
- `scp`로 대상 서버에 jar 전송
- `ssh`로 접속하여 기존 프로세스 종료 후 jar 앱 재실행 처리

---

## 3. 설계 과정

## ✅ `Run Jenkins container (bind-mount)`

```bash
docker run --name myjenkins2 -p 8888:8080 **-v $(pwd)/appjardir:/var/jenkins_home/appjar** jenkins/jenkins:lts-jdk17
```

📦 현재 디렉토리의 `appjardir` 폴더를 컨테이너의 `var/jenkins_home/appjar`에 바인드 마운트합니다.

## ✅ `Jenkins pipeline scripts`

```groovy
pipeline {
    agent any

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/imhaeunim/20250324a.git'
            }
        }
          
        stage('Build') {
            steps {
                dir('./step07_cicd') {                   
                    sh 'chmod +x gradlew'                    
                    sh './gradlew clean build -x test'
                    sh 'echo $WORKSPACE'
                }
            }
        }
        
        stage('Copy jar') { 
            steps {
                script {
                    def jarFile = 'step07_cicd/build/libs/step07_cicd-0.0.1-SNAPSHOT.jar'                   
                    sh "cp ${jarFile} /var/jenkins_home/appjar/"
                }
            }
        }
    }
}

```

🔄 `Clone Repository`

- GitHub 저장소의 `main` 브랜치를 클론합니다.
- 프로젝트 소스를 Jenkins 작업 공간으로 가져옵니다.

🛠 `Build`

- `step07_cicd` 디렉토리로 이동
- Gradle 실행 권한 부여 (`chmod +x gradlew`)
- 테스트 제외하고 빌드 실행 (`./gradlew clean build -x test`)
- `$WORKSPACE` 변수 출력 (Jenkins 작업 공간 경로 확인용)

📦 `Copy jar`

- 빌드된 `.jar` 파일을 Jenkins 컨테이너 내부 경로 `/var/jenkins_home/appjar/`로 복사
- 이 경로는 **호스트 머신과 바운드 마운트**된 디렉토리

## ✅ `inotifywait + sh`

### 🔧 설계 개요

- 호스트에서 직접 jar 파일 변경을 감지
- 변경 시 자동으로 `scp` 전송 및 jar 파일 원격 실행

### 📜 스크립트 예시

```bash
#!/bin/bash

# 감시 대상 JAR 파일 (절대 경로로 설정)
WATCH_FILE="/home/ubuntu/appjardir/step07_cicd-0.0.1-SNAPSHOT.jar"

# 전송 대상 서버 정보
REMOTE_USER="ubuntu"
REMOTE_HOST="myserver02"
REMOTE_PATH="/home/ubuntu/appjardir"

echo "📡 JAR 변경 감지 대기 중... [$WATCH_FILE]"

# 변경 감지 시작
inotifywait -m -e close_write --format "%w%f" "$WATCH_FILE" | while read FILE
do
    echo "📦 JAR 변경 감지됨: $FILE → $REMOTE_USER@$REMOTE_HOST:$REMOTE_PATH"

    # scp로 전송 (StrictHostKeyChecking 비활성화: 자동 전송 목적)
    scp -o StrictHostKeyChecking=no "$FILE" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_PATH/"

     echo "🔁 원격 재시작 시도"
    ssh -o StrictHostKeyChecking=no $REMOTE_USER@$REMOTE_HOST << EOF
    PID=\$(pgrep -f $WATCH_FILE)
    if [ ! -z "\$PID" ]; then
      kill -9 \$PID

      sleep 1
    fi
    nohup java -jar $WATCH_FILE --server.port=8080 > app.log 2>&1 &
EOF
    if [ $? -eq 0 ]; then
        echo "✅ 전송 성공: $FILE → $REMOTE_HOST:$REMOTE_PATH"
    else
        echo "❌ 전송 실패!"
    fi
done

```

📡 `Watch jar`

- `inotifywait`을 사용해 지정한 `.jar` 파일의 변경을 실시간 감지합니다.
- 변경이 감지되면 해당 파일을 원격 서버로 `scp`를 통해 전송합니다.

🔁 `Remote restart`

- 원격 서버에 `ssh`로 접속하여 이전에 실행 중이던 `.jar` 프로세스를 종료합니다.
- 전송된 최신 `.jar` 파일을 `nohup`으로 백그라운드에서 실행합니다.
- 실행 로그는 `app.log`에 저장됩니다.

## ✅ 외부 PC ssh

```bash
#!/bin/bash

# 감시 대상 JAR 파일 (절대 경로로 설정)
WATCH_FILE="/home/ubuntu/appjardir/step07_cicd-0.0.1-SNAPSHOT.jar"

# 상대 파일 경로
TARGET_FILE="/home/ubuntu/yjjar/step07_cicd-0.0.1-SNAPSHOT.jar"

# 전송 대상 서버 정보
REMOTE_USER="ubuntu"
REMOTE_HOST="192.168.0.36"
REMOTE_PATH="/home/ubuntu/yjjar"
REMOTE_PORT="22"

echo "📡 JAR 변경 감지 대기 중... [$WATCH_FILE]"

# 변경 감지 시작
inotifywait -m -e close_write --format "%w%f" "$WATCH_FILE" | while read FILE
do
    echo "📦 JAR 변경 감지됨: $FILE → $REMOTE_USER@$REMOTE_HOST:$REMOTE_PATH"

    # scp로 전송 (StrictHostKeyChecking 비활성화: 자동 전송 목적)
    scp -o StrictHostKeyChecking=no -p "$REMOTE_PORT" "$FILE" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_PATH/"

     echo "🔁 원격 재시작 시도"
    ssh -o StrictHostKeyChecking=no -p "$REMOTE_PORT"  $REMOTE_USER@$REMOTE_HOST << EOF
    PID=\$(pgrep -f $TARGET_FILE)
    if [ ! -z "\$PID" ]; then
      kill -9 \$PID

      sleep 1
    fi
    nohup java -jar $TARGET_FILE --server.port=8088 > app.log 2>&1 &
EOF

    if [ $? -eq 0 ]; then
        echo "✅ 전송 성공: $FILE → $REMOTE_HOST:$REMOTE_PATH"
    else
        echo "❌ 전송 실패!"
    fi
done

```

📦 `Target Info Changed`

- `.jar` 파일의 대상 경로와 실행 파일 경로 변경하였습니다.

🌐 `Remote Host & Port Changed`

- 원격 서버 주소가 `myserver02`에서 `192.168.0.36`으로 변경되었습니다.
- SSH 접속 포트를 명시적으로 `22번`으로 설정하였습니다. (`REMOTE_PORT="22"`)

🚀 `Execution Port Changed`

- Spring Boot 애플리케이션 실행 포트를 `8088`로 변경되었습니다.

## 💡 공개키 전달하는 방법

- SSH 접속시, 비밀번호를 입력해주지 않아도 된다!

<aside>

### SSH key 생성

```yaml
**ssh-keygen -t rsa -b 4096** 
```

### 원격 서버에 SSH 공개 키 추가

SSH 키를 통해 비밀번호 없이 접속할 수 있도록 로컬 서버의 공개 키를 원격 서버에 추가함

공개 키를 원격 서버의 **~/.ssh/authorized_keys 파일에 추가**

```yaml
**ssh-copy-id username@remote_server_ip

#예시 10.0.2.20
ssh-copy-id username@10.0.2.202**
```

### 비밀번호 없이 접속

```yaml
**ubuntu@myserver01:~$ ssh ubuntu@myserver02**
```

</aside>

## 4. 설계 고도화: Jenkins Pipeline 기반 자동화

### 💬 평가

- Jenkins에서 전체 프로세스를 제어함으로써 통합 관리가 용이
- 빌드, 배포, 로그 출력이 한 곳(Jenkins)에서 이루어짐
- 실무적인 구조에 적합하며, 에러 핸들링 및 알림 확장도 용이함

---

✅ **참고 사항**

- Jenkins에서 ssh/scp 사용 시 대상 서버에 공개키 등록 필요
- 포트 번호가 22번이 아닌 경우 `P <포트>` 옵션 추가 필수
- 대상 서버에서 Spring Boot 앱을 서비스로 등록(systemd)하면 더욱 안정적인 재시작 가능
