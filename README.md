# 🚀 Jenkins CICD


## 🎯 프로젝트 목적

- **GitHub에 코드를 push**하면 **Jenkins가 자동으로 빌드**하고 **JAR 파일**을 생성
- 빌드된 JAR은 **Ubuntu 호스트와 Docker 컨테이너 간 Bind Mount**로 공유
- **파일 변경을 감지하는 쉘 스크립트**를 사용하여 JAR파일이 갱신되면 즉시 실행
- 코드 작성부터 빌드와 실행까지 자동으로 이어지는 **CI/CD 실습 구조**를 구축

---

## 🌠 전체 흐름

1. **Spring Application** 코드 작성  
2. **GitHub에 push**  
3. **GitHub Webhook**이 **Jenkins를 자동으로 트리거**
4. Jenkins는 저장소에서 코드를 가져와 빌드하여 **JAR 파일**을 생성
5. 빌드된 JAR 파일은 **Bind Mount**를 통해 **호스트 디렉토리**에 저장 
6. **파일 변경을 감지하는 쉘 스크립트**가 실행되며, JAR 파일이 변경되면 **자동으로 실행**
 
---

## 👥 팀 구성원
<table>
  <tr>
    <td align="center">
      <a href="https://github.com/Minkyoungg0">
        <img src="https://github.com/Minkyoungg0.png" width="100px;" alt="Minkyoungg0"/><br />
        <sub><b>문민경</b></sub>
      </a>
    </td>
    <td align="center">
       <a href="https://github.com/hyunn522">
        <img src="https://github.com/hyunn522.png" width="100px;" alt="hyunn522"/><br />
        <sub><b>정서현</b></sub>
      </a>
    </td>
    <td align="center">
      <a href="https://github.com/moonstone0514">
        <img src="https://github.com/moonstone0514.png" width="100px;" alt="moonstone0514"/><br />
        <sub><b>김문석</b></sub>
      </a>
    </td>
    <td align="center">
      <a href="https://github.com/jihwan77">
        <img src="https://github.com/jihwan77.png" width="100px;" alt="Gill010147"/><br />
        <sub><b>황지환</b></sub>
      </a>
    </td>
    <td align="center">
      <a href="https://github.com/dlacowns21">
        <img src="https://github.com/dlacowns21.png" width="100px;" alt="dlacowns21"/><br />
        <sub><b>임채준</b></sub>
      </a>
    </td>
  </tr>
</table>

---

## 1️⃣ Spring Application 생성 (Gradle)

### a. 예시 코드
```java
@RestController
@RequestMapping("step04")
public class Controller {

    @GetMapping("/get")
    public String getReqRes() {
        return "get 방식 요청의 응답 데이터";
    }

    @PostMapping("/post")
    public String getReqRes2() {
        return "post 방식 요청의 응답 데이터";
    }
}
```

### b. 빌드 & 로컬 실행
```bash
./gradlew clean build

# 실행 테스트
java -jar build/libs/*-SNAPSHOT.jar
curl "http://localhost:8080/app/get"
```

---

## 2️⃣ Jenkins 컨테이너 생성

### a. 호스트 폴더 생성
```bash
# mount할 호스트 디렉토리 생성
mkdir /home/vboxuser/hostvol

# 빌드된 jar 복사
cp build/libs/*.jar ~/hostvol/app.jar
```

### b. Docker로 Jenkins 실행

#### 📌 Bind Mount 개념
> 호스트 디렉토리를 컨테이너 내부 경로에 직접 연결하는 방식  
> 호스트와 컨테이너가 **양방향으로 동일한 디렉토리를 공유**

- Jenkins 컨테이너를 Ubuntu와 **Bind Mount**하려면, 컨테이너 실행 시 `-v` 옵션을 반드시 지정해야 한다.

```bash
docker run --name myjenkins   -v /home/vboxuser/hostvol:/var/jenkins_home/workspace/   -d -p 8080:8080   jenkins/jenkins:lts-jdk17
```

⚠️ 이미 실행 중인 Jenkins 컨테이너에는 새로운 마운트를 추가할 수 없음.  
→ `docker commit`으로 새 이미지를 만든 뒤 실행해야 함.

```bash
# 실행 중인 컨테이너로부터 이미지 생성
docker commit myjenkins myjenkinsimg

# 이미지 확인
docker images

# 새로운 컨테이너 실행
docker run --name myjenkins   -v /home/vboxuser/hostvol:/var/jenkins_home/workspace/   -d -p 8080:8080   myjenkinsimg:latest
```

---

## 3️⃣ Jenkins Project (Pipeline) 생성

```groovy
pipeline {
    agent any

    environment {
        GITHUB_REPO = 'https://github.com/hyunn522/Jenkins_Build.git'
        BRANCH_NAME = 'main'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${BRANCH_NAME}", url: "${GITHUB_REPO}"
                sh 'ls -al'
            }
        }

        stage('Build') {
            steps {
                script {
                    if (fileExists('gradlew')) {
                        sh 'chmod +x gradlew'
                        sh './gradlew build'
                    } else if (fileExists('pom.xml')) {
                        sh 'mvn clean package'
                    } else {
                        error 'Gradle 또는 Maven 프로젝트가 아님'
                    }
                }
            }
        }
    }

    post {
        success { echo '✅ 빌드 성공!' }
        failure { echo '❌ 빌드 실패! 오류 확인 필요!' }
    }
}
```
> 빌드된 JAR : `build/libs/*.jar`  

---

## 4️⃣ GitHub Webhook 설정

### 🔔 Webhook이란?
> 특정 이벤트(GitHub push, Jenkins 빌드 완료 등)가 발생했을 때  
> 미리 정한 URL로 **자동 HTTP 요청을 보내는 방식**
> 이벤트 기반 자동화로, 주기적인 확인(polling) 없이 실시간 처리가 가능

### a. GitHub 저장소 Webhook 등록
<img width="1853" alt="webhook" src="https://github.com/user-attachments/assets/8e9d48b9-d5d2-4fcd-9349-8f9773b4ceeb" />

### b. ngrok을 통한 외부 노출

#### 📌 ngrok 개념
> - **ngrok**은 로컬 서버(예: `http://localhost:8080`)를  
>   외부에서 접근 가능한 **공개 URL**로 매핑해주는 터널링 도구 
> - 방화벽/공유기 설정 없이도 로컬 서버를 인터넷에 노출 가능

---

### 1. ngrok 실행
```bash
ngrok http 8080
```
- Jenkins의 8080 포트를 외부로 노출  
- 실행 후 터미널에 발급된 URL 확인 가능  

<img width="500" alt="ngrok1" src="https://github.com/user-attachments/assets/b40fcbc0-e000-4de5-b208-0da3aa5e7b68" />  <br>
➡ ngrok 실행 후, Jenkins 8080 포트가 외부 주소로 매핑된 화면  

---

### 2. ngrok Public URL 확인
- `https://<랜덤ID>.ngrok.io` 형태의 주소가 발급됨  
- 이 URL을 **GitHub Webhook Payload URL**에 등록  

<img width="500" alt="ngrok2" src="https://github.com/user-attachments/assets/9d805dd0-f40d-4ae8-84bf-af0cd5a304a7" />  <br>
➡ ngrok에서 발급한 외부 접근용 URL 확인  

---

### 3. 브라우저 접속 확인
- 발급받은 ngrok 주소로 접속하면 Jenkins 로그인 화면이 나옴  
- 외부에서도 정상 접근 가능한지 확인 후 Webhook에 등록  

<img width="500" alt="ngrok3" src="https://github.com/user-attachments/assets/28473f4a-205e-458e-83e3-1899a493a3ad" />  <br>
➡ ngrok URL을 통해 Jenkins 로그인 페이지에 접속된 모습  


---

## 5️⃣ 자동화 환경 구성

<img width="5644" alt="flow" src="https://github.com/user-attachments/assets/1525d2c3-f6c0-4ea7-8307-6711949b18be" />

### a. inotifywait 스크립트
```bash
#!/bin/bash

item_name=$1
if [ -z "$item_name" ]; then
    echo "❌ 사용법: $0 <Jenkins 아이템 이름>"
    exit 1
fi

dir_path="/home/vboxuser/hostvol/${item_name}"
if [ ! -d "$dir_path" ]; then
    echo "❌ 디렉토리가 존재하지 않습니다: $dir_path"
    exit 1
fi

inotifywait -m -r -e create,close_write --format '%e %w%f' "$dir_path" |
while read -r events file; do
    if [[ "$file" == *.jar && "$file" != *plain*.jar ]]; then
        echo "🔔 JAR 감지됨: $file"

        pkill -f "java -jar" 2>/dev/null
        sleep 2

        latest_jar=$(ls -t "$dir_path"/build/libs/*.jar | grep -v plain | head -n 1)
        echo "🚀 실행 시작: $latest_jar"
        nohup java -jar "$latest_jar" > "${latest_jar%.jar}.log" 2>&1 &
    fi
done
```

### ✅ 결론

이 스크립트를 통해 **Jenkins 빌드 → JAR 파일 갱신 → 자동 실행**이 완성된다.  
즉, 코드를 **GitHub에 push**하는 순간부터  
애플리케이션이 자동으로 **빌드·실행**되는 **CI/CD 파이프라인**을 구축할 수 있다.

---

## 🛠️ 트러블 슈팅

### 1. 현상
- `docker commit`으로 새 이미지를 만들었음  
- `docker run -v ...` 실행 시, 기존 `/var/jenkins_home/workspace/` 데이터가 보이지 않음  

### 2. 원인
- **바인드 마운트는 호스트 디렉토리를 최우선**으로 사용  
- 컨테이너 경로는 호스트 디렉토리로 완전히 덮어씌워짐  
- 결과적으로 **기존 데이터가 보이지 않는 것처럼 보이는 현상** 발생  

👉 실제로는 지워진 것이 아니라, **호스트 마운트에 가려져 접근할 수 없게 된 것**

