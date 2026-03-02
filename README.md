# 설치 가이드

이 문서는 Podman을 사용하여 OpenWebUi를 설치하고 실오라클 벡터디비 연동하는 방법을 설명합니다.

---

## 목차

1. [사전 준비사항](#1-사전-준비사항)
2. [OCI API Key 설정](#2-oci-api-key-설정)
3. [Podman으로 빠른 시작 (GitHub 이미지)](#3-podman으로-빠른-시작-github-이미지)
4. [OpenWebUI 단독 설치 (Podman)](#4-openwebui-단독-설치-podman)
5. [Podman으로 로컬 이미지 빌드 및 실행](#5-podman으로-로컬-이미지-빌드-및-실행)
6. [환경 변수 설정](#6-환경-변수-설정)
7. [Podman 특화 설정](#7-podman-특화-설정)
8. [애플리케이션 테스트](#8-애플리케이션-테스트)
9. [OpenWebUI 통합 설치](#9-openwebui-통합-설치)
10. [문제 해결](#10-문제-해결)

---

## 1. 사전 준비사항

### 1.1 디렉토리 생성 및 프로젝트 설정 (OCI GenAI + OpenWebUI + Oracle DB)

통합 관리를 위한 폴더 구조:

```bash
# 통합 프로젝트 폴더 생성
mkdir -p oci-ai-stack
mkdir -p .oci                                    # OCI API Key (필수)
mkdir -p wallet                                  # Oracle DB Wallet (선택)

cd ~/oci-ai-stack

# 필수 디렉토리 생성
mkdir -p oci-genai-gateway      # Gateway 설정 및 로그
mkdir -p openwebui         # OpenWebUI 데이터
og
# 선택적 디렉토리 (필요시 생성)
# mkdir -p models                                # 로컬 모델 파일
# mkdir -p certs                                 # SSL 인증서

# 전체 구조 확인
tree ~/oci-ai-stack -L



# 전체 구조 확인
tree ~/oci-ai-stack


### 1.1.0 Git 설치 (필수)

`git clone`을 사용하려면 Git이 필요합니다. 시스템에 Git이 없다면 다음 명령어로 설치해 주세요:

```bash
sudo dnf install -y git
```

### 1.1.1 GitHub에서 초기 코드 다운로드

프로젝트를 직접 실행하거나 커스텀하기 위해 기본 소스 코드를 GitHub에서 가져옵니다. `~/oci-ai-stack` 안에서 다음 명령으로 복제하세요:

```bash
git clone https://github.com/jin38324/OCI_GenAI_access_gateway.git ~/oci-ai-stack/OCI_GenAI_access_gateway
```

`OCI_GenAI_access_gateway/` 내부에 `app/`, `deployments/`, `requirements.txt` 등 실행에 필요한 파일들이 위치합니다. 이후 단계에서는 이 디렉토리와 `~/oci-ai-stack`을 기준으로 설명합니다.

※ 원본 repo에는 `.env` 파일이 포함되지 않으므로, `oci-genai-gateway`의 환경 변수는 쉘에서 `export`하는 방식으로, OpenWebUI는 별도 `.env`를 만들어 관리하는 방식으로 설정합니다. `~/oci-ai-stack` 루트 안에 `oci-genai-gateway/`(설정, logs)와 `openwebui/`(config, data, .env), `.oci/`, `wallet/` 등을 두면 Podman Compose나 개별 실행할 때 경로를 명확하게 유지할 수 있습니다.


### 1.1.2 현실에 맞는 ~/oci-ai-stack 구조

현재 클론된 저장소 구조는 `~/oci-ai-stack/OCI_GenAI_access_gateway` 하나지만, 실제로는 `~/oci-ai-stack`를 루트로 삼아 다음 구조를 만들어 사용한다고 상상해보시면 됩니다:

```text
~/oci-ai-stack/
├── OCI_GenAI_access_gateway/   # GitHub에서 받은 소스 (app/, deployments/ 등)
├── openwebui/                 # OpenWebUI config/data/logs/.env
├── .oci/                      # OCI API Key (컨테이너 /root/.oci/ 경로에 매핑)
└── wallet/                    # Oracle DB Wallet (필요 시)
```

`OCI_GenAI_access_gateway/` 자체는 Git 저장소로 관리하고, 나머지 디렉토리는 Podman 실행 및 환경 변수 `.env`를 담는 공간으로 사용하는 패턴입니다. 이 구조는 실제 클론한 경로(`~/oci-ai-stack/OCI_GenAI_access_gateway`)를 기준으로 그대로 적용 가능합니다.
```

**권장 폴더 구조**(실제 클론된 `OCI_GenAI_access_gateway/`를 포함한 구성으로 현실화):
```
~/oci-ai-stack/
├── OCI_GenAI_access_gateway/   # Git에서 받은 소스(app/, deployments/, README 등)
├── openwebui/                 # OpenWebUI config/data/logs 및 자체 .env
├── .oci/                      # OCI API Key (컨테이너 /root/.oci/에 매핑)
├── wallet/                    # Oracle DB Wallet (선택)
├── models/                    # 로컬 모델 파일 (선택)
├── certs/                     # SSL 인증서 (선택)
└── podman-compose.yaml        # 통합 Compose 파일 (선택)
```

`OCI_GenAI_access_gateway/`는 그대로 Git 저장소로 유지하며, 나머지 디렉토리를 `~/oci-ai-stack` 루트에 추가로 만들어 Podman 실행 시 고정된 볼륨 경로로 사용하는 방식을 권장합니다. `.env` 파일이 없는 gateway는 쉘 환경 변수로, OpenWebUI는 별도 `.env` 파일로 관리해도 됩니다.

### 1.2 Podman 설치 (Oracle Linux ARM 기준)

**Oracle Linux 8/9 (ARM64/aarch64)**:

```bash
# 1. 시스템 업데이트
sudo dnf update -y

# 2. Podman 및 필수 패키지 설치
sudo dnf install -y podman podman-docker containernetworking-plugins

# 3. Podman Compose 설치 (필요한 경우)
#sudo dnf install -y podman-compose

# 4. 설치 확인
    # 시스템 부팅 시 자동 시작하려면:
    #systemctl --user enable podman.socket
    #systemctl --user start podman.socket
    ```

**Podman 버전 확인**:

```bash
# 버전 확인
podman version

# Podman 정보 확인 (아키텍처 포함)
podman info

# ARM 아키텍처 확인
uname -m  # aarch64 출력 확인
```

### 1.2.1 방화벽 및 포트 준비

OCI GenAI Gateway(8088)와 OpenWebUI(3000)를 외부에서 접근 가능하게 하려면 방화벽에서 해당 포트를 열어야 합니다. 예를 들어:

```bash
sudo firewall-cmd --permanent --add-port=8088/tcp
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --reload
```

### 1.3 필수 파일 확인
```

### 1.2 필수 파일 확인

프로젝트 디렉토리 구조:
```
OCI_GenAI_access_gateway/
├── Dockerfile
├── requirements.txt
├── app/
│   ├── app.py
│   ├── config.py
│   ├── models.yaml
│   └── ...
└── deployments/
```

---

## 2. OCI API Key 설정

### 2.1 OCI 콘솔에서 API Key 생성

1. [OCI Console](https://cloud.oracle.com/)에 로그인
2. **User Settings** → **API Keys** 클릭
3. **Add API Key** → **Generate API Key Pair** 선택
4. Private Key와 Public Key를 다운로드

### 2.2 로컬 설정 파일 구성

OCI CLI 또는 SDK 설정 파일은 로컬 홈(`~/.oci`)에 생성한 뒤, Podman에서 사용할 수 있도록 `~/oci-ai-stack/.oci`로 복사합니다. 이 과정을 통해 호스트에서 안전하게 관리하는 `~/.oci` 안의 민감한 키를 컨테이너가 읽을 수 있게 마운트된 경로로 이동시키며, 컨테이너 내부에서 `~/oci-ai-stack/.oci`를 `/root/.oci`로 붙이면 OCI SDK가 올바른 경로(`/root/.oci/config`, `/root/.oci/oci_api_key.pem`)를 그대로 참조할 수 있습니다. `~/.oci/config`에는 다음과 같이 OCI 자격증명을 입력하고, `key_file` 경로는 컨테이너 내부 `/root/.oci/oci_api_key.pem`을 가리키도록 작성하세요.
```
# OCI 설정 디렉토리 생성
mkdir -p ~/.oci

# 다운로드한 키 파일 이동
mv ~/Downloads/oci_api_key.pem ~/.oci/

# 설정 파일 생성
cat > ~/.oci/config << 'EOF'
[DEFAULT]
user=ocid1.user.oc1..xxxxx
fingerprint=xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx
tenancy=ocid1.tenancy.oc1..xxxxx
region=us-chicago-1
key_file=/root/.oci/oci_api_key.pem
EOF

# 파일 권한 설정 (보안)
chmod 600 ~/.oci/config
chmod 600 ~/.oci/oci_api_key.pem
```
```

Podan으로 컨테이너를 띄울 때 `~/oci-ai-stack/.oci`를 `/root/.oci`로 마운트합니다. 예: `-v ~/oci-ai-stack/.oci:/root/.oci:Z`. 이렇게 하면 컨테이너 안에서 `~/.oci/config`가 존재하고, `key_file` 경로도 내부 `/root/.oci/oci_api_key.pem`으로 일치합니다.

```bash
# 작업 디렉토리 안에 .oci가 없으면 생성
mkdir -p ~/oci-ai-stack/.oci

# 홈의 ~/.oci 내용을 작업 디렉토리로 복사
cp -r ~/.oci/* ~/oci-ai-stack/.oci/
```

---

## 3. Podman으로 빠른 시작 (GitHub 이미지)

GitHub Container Registry에서 미리 빌드된 이미지를 사용하여 바로 실행합니다.

### 3.1 models.yaml과 환경 변수 일치
`app/config.py`도 기본값을 담고 있으므로, 수정하거나 오버라이드할 값을 식별해 두면 좋습니다. 예:

```
PORT = 8088
AUTH_TYPE = "INSTANCE_PRINCIPAL"  # 또는 API_KEY
DEFAULT_API_KEYS = "ocigenerativeai"
OCI_CONFIG_FILE = "~/.oci/config"
OCI_CONFIG_FILE_KEY = "DEFAULT"
REGION = ""  # 실행 환경에서 채우는 값
OCI_COMPARTMENT = ""
```

`REGION`/`OCI_COMPARTMENT`을 빈 문자열로 두면, 컨테이너 시작 시 지정한 환경 변수(`OCI_REGION`, `OCI_COMPARTMENT`)로 채워집니다. `AUTH_TYPE`을 `INSTANCE_PRINCIPAL`로 설정하면 API Key를 사용하지 않으며, `API_KEY`로 변경할 경우 Gateway에 API 키를 직접 전달할 수 있다는 점도 문서에 명시해 두면 좋습니다.


`app/models.yaml`의 최상단 `region`/`compartment_id` 값은 Gateway가 호출하는 OCI 리전 및 컴파트먼트를 고정합니다. 아래 예시와 같이 실제 사용할 값으로 설정하고, 쉘 환경 변수(`OCI_REGION`, `OCI_COMPARTMENT`)가 동일한 값을 갖도록 맞춰야 합니다.

```yaml
- region: us-chicago-1
  compartment_id: ocid1.compartment.oc1..aaaaaaaamhnnkllyjcdshpckz35o7w2lq5sylem33wttw7xj6kvg6bwex5pq
  models:
    ...
```

```bash
# models.yaml과 동일한 리전 및 컴파트먼트
# OCI_REGION/OCI_COMPARTMENT은 models.yaml에 정의된 값과 일치하도록만 유지하면 됩니다.
```

### 3.2 컨테이너 실행

```bash

## API Key 
podman run -d \
    --name oci-genai-gateway \
    -p 8088:8088 \
    -v ~/oci-ai-stack/OCI_GenAI_access_gateway/app:/app:Z \
    -v ~/oci-ai-stack/.oci:/root/.oci:Z \
    ghcr.io/jin38324/oci-genai-access-gateway:v20251217

## Instance Principal 
podman run -d \
    --name oci-genai-gateway \
    -p 8088:8088 \
    -v /home/opc/oci-ai-stack/OCI_GenAI_access_gateway/app:/app:Z \
    ghcr.io/jin38324/oci-genai-access-gateway:v20251217


```

`podman run`은 지정한 이미지(이 예에서는 `ghcr.io/jin38324/oci-genai-access-gateway:v20251217`)를 새 컨테이너로 실행하는 명령입니다. `podman start`는 이미 생성된(생성을 위해 `podman run` 또는 `podman create`로 만든) 컨테이너를 재시작할 때만 사용합니다. 따라서 첫 실행이나 이미지가 변경된 경우에는 `podman run`을 다시 실행해 새로운 컨테이너를 만들고, 이후에는 `podman start oci-genai-gateway`/`podman stop oci-genai-gateway`로 컨테이너를 절전/재시작할 수 있습니다.

이미지를 완전히 다시 만들고 싶다면 `podman rm oci-genai-gateway`로 기존 컨테이너를 제거한 뒤 `podman rmi ghcr.io/jin38324/oci-genai-access-gateway:v20251217`로 이미지를 지우고 `podman run`을 다시 실행하면 새로운 이미지 레이어를 풀 또는 빌드하여 컨테이너를 만들 수 있습니다.

**Podman 특수 파라미터 설명**:
- `:Z` - SELinux 볼륨 마운트 레이블링 (rootless Podman에서 필수)

### 3.3 컨테이너 상태 확인

```bash
# 실행 중인 컨테이너 목록
podman ps

# 로그 확인
podman logs -f oci-genai-gateway

# 컨테이너 내부 접속 (디버깅용)
podman exec -it oci-genai-gateway /bin/bash
```

`podman ps`로 실행 중인 컨테이너가 보이지 않거나 `Exited` 상태라면 `podman ps -a`로 상태를 확인하고 `podman logs -f oci-genai-gateway`를 사용해 오류를 확인하세요. 로그를 통해 `InvalidConfig` 등이 나타나면 OCI 설정(fingerprint, user, tenancy, key_file)과 `/root/.oci/config` 위치를 다시 점검합니다.

또한 컨테이너가 올라온 후에는 `curl http://localhost:8088/docs` 또는 `curl http://IP:8088/docs`로 API 문서가 열리는지 테스트하여 접속성도 확인하세요.

---

## 4. OpenWebUI 단독 설치 (Podman)

OpenWebUI를 OCI GenAI Gateway와 연동하여 사용하려면, 먼저 OpenWebUI 컨테이너를 실행합니다.

### 4.1 OpenWebUI 준비

```bash
# OpenWebUI를 GitHub에서 받아옵니다
git clone https://github.com/open-webui/open-webui.git ~/oci-ai-stack/openwebui
cd ~/oci-ai-stack/openwebui
```

> **Podman 사용자라면** 아래 로컬 빌드/백엔드 단계는 선택 사항입니다. `ghcr.io/open-webui/open-webui:main` 이미지는 이미 프론트엔드 번들 및 Python 백엔드를 포함하고 있으므로, 컨테이너만으로 충분합니다. 다만 커스텀 빌드나 직접 실행이 필요할 때는 참고하세요.

#### (선택) 프론트엔드 로컬 빌드

```bash
# Node >= 18.13.0 (<=22.x.x), npm >= 6 필요
# 예: Oracle Linux에서
# curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo -E bash -
# sudo dnf install -y nodejs

npm install -f
npm run build
```

#### (선택) Backend (Python) 준비

```bash
cd ~/oci-ai-stack/openwebui/backend

# Python 3.10+ 권장, 가상환경 생성
python3 -m venv .venv
source .venv/bin/activate

# 의존성 설치 (requirements.txt / requirements-min.txt 중 선택)
pip install --upgrade pip
pip install -r requirements.txt

# 환경 변수(.env) 준비 후 백엔드 실행
./start.sh
# 또는 Windows: start_windows.bat

# 개발 모드 실행 스크립트가 필요하면 ./dev.sh 참조
```

#### 4.1.1 환경 변수 파일 (.env) 작성

OpenWebUI는 기본적으로 OpenAI 호환 API 엔드포인트를 기대합니다. OCI GenAI Gateway에 연결하기 위해 `~/oci-ai-stack/openwebui/.env` 파일을 직접 만들어 필요한 값을 정의합니다.

```bash
cat > .env <<'EOF'
###### OpenWebUI 설정

# oci genai access gateway 설정

ENABLE_OLLAMA_API=False
# In docker-compose.yml or environment variables
# OPENWEBUI_PIPELINE_URL=http://localhost:9099

# OPENAI_API_BASE_URL=http://difychat.duckdns.org:8088/v1
# OPENAI_API_BASE_URL=http://localhost:8088/v1
OPENAI_API_BASE_URL=http://host.containers.internal:8088/v1
OPENAI_API_KEY=ocigenerativeai
#RAG_OPENAI_API_BASE_URL=http://localhost:8088/v1
RAG_OPENAI_API_BASE_URL=http://host.containers.internal:8088/v1
RAG_OPENAI_API_KEY=ocigenerativeai
RAG_EMBEDDING_ENGINE=openai
RAG_EMBEDDING_MODEL=cohere.embed-v4.0
RAG_EMBEDDING_MODEL_AUTO_UPDATE=false
RAG_EMBEDDING_MODEL_TRUST_REMOTE_CODE=true

# ORACLE23AI (Oracle23ai Vector Search) 설정
VECTOR_DB=oracle23ai
# VECTOR_DB=chroma

# Oracle ADB Connection Settings
ORACLE_DB_USE_WALLET=true
ORACLE_DB_USER=OPENWEBUIUSER
ORACLE_DB_PASSWORD=Oracle_12345
ORACLE_DB_DSN=(description= (retry_count=20)(retry_delay=3)(address=(protocol=tcps)(port=1522)(host=adb.ap-seoul-1.oraclecloud.com))(connect_data=(service_name=lowxxxxxraclejason_low.adb.oraclecloud.com))(security=(ssl_server_dn_match=yes)))
ORACLE_WALLET_DIR=/app/wallet
ORACLE_WALLET_PASSWORD=Oracle_12345
ORACLE_VECTOR_LENGTH=1024

# DB POOL
ORACLE_DB_POOL_MIN=2
ORACLE_DB_POOL_MAX=10
ORACLE_DB_POOL_INCREMENT=1

# debug option
GLOBAL_LOG_LEVEL=DEBUG
EOF
```

> **TIP**: `host.containers.internal`은 Podman rootless 모드에서 호스트 네트워크를 가리키는 예약 호스트명입니다. 만약 OpenWebUI가 다른 머신에 있고 OCI Gateway에 네트워크로 접근해야 한다면, `OPENAI_API_BASE_URL` 값을 `http://<게이트웨이 IP>:8088/v1` 형식으로 교체하세요.

주요 변수 설명:

| 변수 | 설명 |
|------|------|
| `ENABLE_OLLAMA_API` | 내부 Ollama 파이프라인 사용 여부. OCI Gateway만 사용할 경우 `False` 유지 |
| `OPENWEBUI_PIPELINE_URL` | Docker Compose 등에서 별도 파이프라인이 있을 때만 주석 해제하고 사용 |
| `OPENAI_API_BASE_URL` / `OPENAI_API_KEY` | OCI GenAI Gateway OpenAI 호환 엔드포인트/키. 로컬 게이트웨이 주소를 사용하지 않는다면 실제 경로로 수정 |
| `RAG_*` 항목 | RAG 파이프라인 임베딩 엔진/모델/키. Cohere 임베딩 모델을 OCI Gateway 경유로 호출하는 구성을 전제로 함 |
| `VECTOR_DB` | `oracle23ai` 로 설정하면 Oracle 23ai Vector Search(ADB)를 사용. 필요시 주석의 chroma 등 다른 스토어로 전환 |
| `ORACLE_DB_*` / `ORACLE_WALLET_*` | Autonomous Database 접속 정보와 Wallet 경로/암호. 환경에 맞게 사용자/비밀번호/DSN/Wallet 경로 수정 |
| `ORACLE_DB_POOL_*` | Oracle 커넥션 풀 최소/최대/증분 설정 |
| `GLOBAL_LOG_LEVEL` | OpenWebUI 백엔드 로그 레벨 |

필요하면 `.env`에 추가 변수를 정의해 Podman 실행 시 자동으로 주입할 수 있습니다. (예: `ENABLE_SIGNUP`, `WEBUI_SECRET_KEY`, `TZ`, `PROXY_URL` 등)

#### 4.1.2 Vector DB (Oracle) 연동 변수

Oracle Database를 벡터 저장소로 사용할 예정이라면, Oracle Database Wallet과 접속 정보를 컨테이너에 전달해야 합니다. 기본 구조에서 `~/oci-ai-stack/wallet` 디렉토리를 준비했다면 다음 변수를 `.env`에 추가하세요.

```bash
cat >> .env <<'EOF'
# Oracle Vector DB 설정
VECTOR_STORE=oracle
ORACLE_DB_DSN=MYDB_high            # tnsnames.ora에 정의된 서비스 이름
ORACLE_DB_USER=openwebui_vector
ORACLE_DB_PASSWORD='strong_password'
TNS_ADMIN=/app/wallet              # 컨테이너 내부 Wallet 경로
EOF
```

Oracle Instant Client Wallet 파일은 컨테이너에 마운트해야 하므로, 이후 Podman 실행 시 `-v ~/oci-ai-stack/wallet:/app/wallet:Z` 옵션을 추가합니다. `ORACLE_DB_DSN`, `ORACLE_DB_USER`, `ORACLE_DB_PASSWORD` 값은 실제 DB 접속 정보에 맞춰 수정하세요.

### 4.2 OpenWebUI 실행 (OCI Gateway 연동)

`.env`를 준비했다면 Podman에서 해당 파일을 불러와 컨테이너를 실행합니다.

```bash
podman run -d \
    --name openwebui \
    --env-file .env \
    -p 3000:8080 \
    -v ~/oci-ai-stack/openwebui:/app/backend/data:Z \
    -v /home/opc/wallet:/app/wallet:Z \
    ghcr.io/open-webui/open-webui:main
```
> **데이터 보존 안내**
> - 컨테이너 내부 기본 경로는 `/app/backend/data`, `/app/config`, `/app/logs`입니다. Podman 명령에 있는 `-v ~/oci-ai-stack/openwebui/{data,config,logs}:...` 옵션은 이 경로들을 호스트 디렉토리와 연결해 영구 저장하려는 경우에만 필요합니다.
> - 이미 `~/oci-ai-stack/openwebui/backend/data` 등 로컬 백엔드 디렉토리를 운영 중이라면 해당 경로를 그대로 볼륨에 매핑해도 됩니다(예: `-v ~/oci-ai-stack/openwebui/backend/data:/app/backend/data:Z`).
> - 반대로, 데이터를 호스트에 남기지 않아도 된다면 해당 `-v` 옵션을 제거하고 컨테이너 내부 스토리지를 사용해도 됩니다. 다만 Wallet은 필수이므로 `/home/opc/wallet` 디렉터리는 반드시 만들어야 하며(예: `mkdir -p /home/opc/wallet`), 실제 Wallet 위치에 맞춰 `-v <경로>:/app/wallet:Z`를 조정하세요.


Wallet을 아직 준비하지 않았다면 `/home/opc/wallet`에 필요한 파일을 복사하거나, 실제 Wallet 경로에 맞춰 `-v <경로>:/app/wallet:Z` 옵션을 수정하세요.
> **주의**: `/home/opc/wallet` 디렉토리가 존재해야 합니다. Wallet을 아직 준비하지 않았다면 `mkdir -p /home/opc/wallet` 후 필요한 파일을 복사하거나, 실제 Wallet 경로에 맞춰 `-v <경로>:/app/wallet:Z` 옵션을 수정하세요.

만약 OpenWebUI를 다른 네트워크에서 실행하여 OCI GenAI Gateway의 호스트 IP를 직접 지정해야 한다면, `.env` 안의 `OPENAI_API_BASE_URL`만 변경하면 됩니다.

```bash
sed -i "s#host.containers.internal:8088#${GATEWAY_IP:-192.168.0.10}:8088#" .env
```

컨테이너가 올라온 뒤 브라우저에서 `http://<호스트 IP>:3000`으로 접속해 초기 관리자 계정을 생성하고, **Settings → Connections → OpenAI API** 값이 `.env` 내용과 일치하는지 확인합니다.



### 9.7 접속 정보

| 서비스 | URL | 설명 |
|--------|-----|------|
| OCI GenAI Gateway | http://localhost:8088 | API 엔드포인트 |
| OpenWebUI | http://localhost:3000 | 웹 채팅 인터페이스 |
| API Docs | http://localhost:8088/docs | Swagger UI |

### 9.8 OpenWebUI 초기 설정

1. 브라우저에서 http://localhost:3000 접속
2. 첫 로그인 시 관리자 계정 생성
3. **Settings** → **Connections** → **OpenAI API** 확인
   - URL: `http://oci-genai-gateway:8088/v1` (자동 설정됨)
   - Key: `ocigenerativeai` (자동 설정됨)
4. **New Chat**에서 OCI GenAI 모델 선택 후 사용

### 9.9 개별 실행/중지


---

## 10. 문제 해결

### 10.1 일반적인 문제

#### 문제: `Permission denied` 오류

**원인**: SELinux 또는 파일 권한 문제

**해결책**:
```bash
# SELinux 레이블 재설정
restorecon -R ~/.oci/

# 볼륨 마운트 시 :Z 플래그 확인
podman run ... -v ~/.oci:/root/.oci:Z ...
```

#### 문제: `OCIRegionNotExists` 또는 인증 오류

**원인**: OCI 설정 파일 경로 또는 환경 변수 문제

**해결책**:
```bash
# 컨테이너 내부에서 설정 파일 확인
podman exec -it oci-genai-gateway /bin/bash
cat /root/.oci/config

# 환경 변수 확인
echo $OCI_REGION
echo $OCI_COMPARTMENT
```

#### 문제: "Compartment ID must be provided" 에러

**원인**: `models.yaml`에 설정된 `compartment_id`가 API 호출 시 전달되지 않음

**증상**: 
- cohere 모델은 정상 작동하지만 meta, xai, openai 등 다른 모델 호출 시 에러 발생
- 로그에 `ResponseValidationError: 5 validation errors` 표시

**해결책**:
`app/api/models/oci_chat.py` 파일을 수정하여 모델별 `compartment_id`를 사용하도록 변경:

```bash
# 컨테이너 중지
podman stop oci-genai-gateway
podman rm oci-genai-gateway

# oci_chat.py 파일 수정 (149-151번 줄)
# 수정 전:
#   http_client_headers = self._build_headers(compartment_id = OCI_COMPARTMENT)
# 수정 후:
#   model_info = SUPPORTED_OCIGENAI_CHAT_MODELS[self.model_id]
#   compartment_id = model_info.get("compartment_id", OCI_COMPARTMENT)
#   http_client_headers = self._build_headers(compartment_id = compartment_id)

# _invoke_response 메서드도 동일하게 수정 (233-235번 줄)
# 수정 전:
#   http_client_headers = self._build_headers(OCI_COMPARTMENT, chat_request.get("conversation"))
# 수정 후:
#   model_info = SUPPORTED_OCIGENAI_CHAT_MODELS[self.model_id]
#   compartment_id = model_info.get("compartment_id", OCI_COMPARTMENT)
#   http_client_headers = self._build_headers(compartment_id, chat_request.get("conversation"))

# 컨테이너 재시작
podman run -d \
    --name oci-genai-gateway \
    -p 8088:8088 \
    -v /home/opc/oci-ai-stack/OCI_GenAI_access_gateway/app:/app:Z \
    localhost/oci-genai-access-gateway:local
```

**참고**: 
- `config.py`의 `OCI_COMPARTMENT`가 빈 문자열이면 `models.yaml`에서 모델 설정을 읽어옵니다
- 각 모델별로 `compartment_id`가 `models.yaml`에 정의되어 있어야 합니다
- 수정 후 반드시 컨테이너를 재시작해야 변경사항이 적용됩니다

#### 문제: 포트 이미 사용 중

**해결책**:
```bash
# 사용 중인 포트 확인
sudo ss -tlnp | grep 8088

# 기존 컨테이너 중지 및 삭제
podman stop oci-genai-gateway
podman rm oci-genai-gateway

# 다른 포트 사용
podman run ... -p 8089:8088 ...
```

### 10.2 Podman 특정 문제

#### 문제: 이미지 pull 실패

**해결책**:
```bash
# GitHub Container Registry 로그인 (필요한 경우)
podman login ghcr.io

# 또는 이미지 수동 다운로드 후 로드
podman pull ghcr.io/jin38324/oci-genai-access-gateway:v20251217
```

#### 문제: rootless 포트 바인딩 제한

**원인**: Rootless 모드에서 1024 미만 포트는 특권 필요

**해결책**:
```bash
# 1024 이상 포트 사용 (예: 8088)
podman run ... -p 8088:8088 ...

# 또는 unprivileged_port_start 설정 변경
sudo sysctl net.ipv4.ip_unprivileged_port_start=80
```

### 10.3 로그 확인 및 디버깅

```bash
# 실시간 로그 확인
podman logs -f oci-genai-gateway

# 마지막 100줄 로그 확인
podman logs --tail 100 oci-genai-gateway

# 컨테이너 내부 파일 확인
podman exec oci-genai-gateway ls -la /app/

# 네트워크 연결 테스트
podman exec oci-genai-gateway curl -v https://generativeai.${OCI_REGION}.oci.oraclecloud.com
```

### 10.4 OpenWebUI 특정 문제

#### 문제: OpenWebUI에서 모델 목록이 안 보임

**해결책**:
```bash
# OCI GenAI Gateway가 정상 실행 중인지 확인
podman-compose ps

# OpenWebUI 컨테이너에서 Gateway 연결 테스트
podman exec openwebui curl http://oci-genai-gateway:8088/v1/models

# OpenWebUI 로그 확인
podman-compose logs openwebui
```

#### 문제: OpenWebUI에서 "Connection refused" 오류

**해결책**:
```bash
# 네트워크 확인
podman network ls
podman network inspect ai-stack-network

# 두 컨테이너가 같은 네트워크에 있는지 확인
podman inspect oci-genai-gateway | grep -A 5 Networks
podman inspect openwebui | grep -A 5 Networks
```

### 10.5 Oracle DB Wallet 특정 문제

#### 문제: Wallet 파일 접근 권한 오류

**원인**: SELinux 또는 파일 권한 문제

**해결책**:
```bash
# Wallet 디렉토리 SELinux 컨텍스트 확인
ls -Z ~/oci-ai-stack/wallet/

# SELinux 레이블 재설정
restorecon -R ~/oci-ai-stack/wallet/

# 또는 permissive 모드로 변경 (임시)
sudo setenforce 0
```

#### 문제: TNS_ADMIN 환경 변수 오류

**해결책**:
```bash
# 컨테이너 내부에서 환경 변수 확인
podman exec openwebui echo $TNS_ADMIN

# 컨테이너 내부에서 Wallet 파일 확인
podman exec openwebui ls -la /app/wallet/

# tnsnames.ora 파일 확인
podman exec openwebui cat /app/wallet/tnsnames.ora
```

#### 문제: Oracle DB 연결 실패

**해결책**:
```bash
# Wallet 파일 존재 확인
ls -la ~/oci-ai-stack/wallet/

# 파일 권한 확인 (읽기 가능해야 함)
chmod 644 ~/oci-ai-stack/wallet/*

# 컨테이너에서 직접 테스트 (sqlplus 또는 cx_Oracle 필요)
podman exec -it openwebui /bin/bash
# 내부에서: sqlplus user/password@MYDB_high
```

### 10.6 컨테이너 정리

```bash
# 실행 중인 컨테이너 중지
podman stop oci-genai-gateway

# 컨테이너 삭제
podman rm oci-genai-gateway

# 이미지 삭제 (재설치 시)
podman rmi oci-genai-gateway:local

# 전체 정리 (미사용 리소스)
podman system prune -f
```

---

## 참고 자료

- [OCI Generative AI Documentation](https://docs.oracle.com/en-us/iaas/Content/generative-ai/home.htm)
- [OCI SDK Configuration](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/sdkconfig.htm)
- [Podman Documentation](https://docs.podman.io/)
- [GitHub Repository](https://github.com/jin38324/OCI_GenAI_access_gateway)

---

## 지원 모델

`models.yaml`에서 설정 가능한 모델 유형:

1. **ondemand**: OCI Generative AI 사전 학습 챗 모델
2. **embedding**: OCI Generative AI 임베딩 모델
3. **dedicated**: 전용 AI 클러스터 모델
4. **datascience**: OCI Data Science AI Quick Action 배포 모델
5. **imported**: 가져온 모델
6. **grok**: XAI Grok 모델
7. **gemini**: Google Gemini 모델
8. **openai**: OpenAI GPT 모델

---

**문서 버전**: 2025.02  
**Podman 버전**: 4.x 이상 권장  
**OCI GenAI Gateway 버전**: v20251217

## 5. MCPO + SQLcl MCP Server 설치

  MCPO(MCP-to-OpenAPI Proxy)를 설치하여 SQLcl의 내장 MCP Server를 HTTP OpenAPI 엔드포인트로
  변환합니다.
  OpenWebUI 등 다른 도구에서 SQLcl의 데이터베이스 기능을 REST API로 호출할 수 있게 됩니다.

  ### 5.1 개요

  [SQLcl MCP Server] ── stdio ──▶ [MCPO] ── HTTP/OpenAPI ──▶ [OpenWebUI 등 클라이언트]
          (포트 없음)                (포트 8000)                   (포트 3000)

  | 구성 요소 | 역할 |
  |-----------|------|
  | SQLcl (25.1+) | Oracle DB 접속 및 SQL 실행 (MCP Server 내장) |
  | MCPO | MCP 프로토콜을 OpenAPI(HTTP)로 변환하는 프록시 |
  | Oracle Wallet | DB 인증 정보 (비밀번호 노출 없이 접속) |

  ### 5.2 사전 준비사항

  #### 5.2.1 Oracle JDK 설치

  SQLcl은 Java가 필요합니다. Oracle JDK를 설치합니다 (OpenJDK 아님):

  ```bash
  # Oracle JDK 17 설치
  sudo rpm -ivh https://download.oracle.com/java/17/latest/jdk-17_linux-x64_bin.rpm

  # 설치 확인 (Oracle 표시 확인)
  java -version
  # java version "17.x.x"
  # Java(TM) SE Runtime Environment (build 17.x.x)
  # Java HotSpot(TM) 64-Bit Server VM
  ```

  5.2.2 SQLcl 설치

  ```bash
  # SQLcl 설치
  sudo dnf install sqlcl

  # 설치 확인 (25.1 이상 필요)
  sql -version

  # 설치 경로 확인
  which sql
  rpm -ql sqlcl
  ```

  참고: SQLcl 25.1 버전부터 MCP Server 기능이 내장되어 있습니다. 별도 빌드나 추가 설치 없이
  sql /nolog @mcp 명령으로 MCP Server 모드를 실행할 수 있습니다.

  5.2.3 uv 설치

  MCPO를 가상환경 없이 간편하게 실행하기 위해 uv를 사용합니다:

  ```bash
  # uv 설치
  curl -LsSf https://astral.sh/uv/install.sh | sh
  source ~/.bashrc

  # 설치 확인
  uv --version
  ```

# 설치 확인
  uv --version

  5.3 MCPO 설정

  5.3.1 디렉토리 생성

  mkdir -p ~/oci-ai-stack/mcpo

  5.3.2 config.json 생성

  Oracle Wallet을 사용하여 비밀번호 노출 없이 DB에 접속하도록 설정합니다:

  cat > ~/oci-ai-stack/mcpo/config.json << 'EOF'
  {
    "mcpServers": {
      "sqlcl": {
        "command": "sql",
        "args": ["/nolog", "@mcp"],
        "env": {
          "TNS_ADMIN": "/home/opc/wallet"
        }
      }
    }
  }
  EOF

  설명:
  - "command": "sql" — SQLcl 실행 파일
  - "args": ["/nolog", "@mcp"] — DB 미접속 상태로 MCP Server 모드 시작
  - "TNS_ADMIN": "/home/opc/wallet" — 기존 Oracle Wallet 경로를 지정하여 Wallet 인증 사용

  5.3.3 폴더 구조 확인

  ~/oci-ai-stack/
  ├── OCI_GenAI_access_gateway/   # 기존 (포트 8088)
  ├── openwebui/                  # 기존 (포트 3000)
  ├── .oci/                       # 기존 (OCI API Key)
  ├── wallet/                     # 기존 (Oracle Wallet - SQLcl 인증에 사용)
  └── mcpo/                       # 새로 추가 (포트 8000)
      └── config.json

  5.4 방화벽 설정

  sudo firewall-cmd --permanent --add-port=8000/tcp
  sudo firewall-cmd --reload

  5.5 MCPO 실행

  cd ~/oci-ai-stack/mcpo
  uvx mcpo --port 8000 --api-key "
  
  " --config ./config.json

  TIP: uvx는 MCPO를 자동으로 다운로드하고 격리된 환경에서 실행합니다. 별도로 pip install이나
  가상환경(venv) 설정이 필요 없습니다.

  5.6 동작 확인

  브라우저에서 접속:
  http://<서버IP>:8000/docs

  자동 생성된 OpenAPI 문서에서 SQLcl MCP 도구(execute_sql, list_connections 등)가 보이면
  성공입니다.

  5.7 전체 서비스 포트 정리

  ┌────────────────────┬───────────┬────────────────────────────┐
  │       서비스       │   포트    │            용도            │
  ├────────────────────┼───────────┼────────────────────────────┤
  │ OCI GenAI Gateway  │ 8088      │ LLM API 엔드포인트         │
  ├────────────────────┼───────────┼────────────────────────────┤
  │ OpenWebUI          │ 3000      │ 웹 채팅 인터페이스         │
  ├────────────────────┼───────────┼────────────────────────────┤
  │ MCPO               │ 8000      │ SQLcl MCP → OpenAPI 프록시 │
  ├────────────────────┼───────────┼────────────────────────────┤
  │ API Docs (Gateway) │ 8088/docs │ Gateway Swagger UI         │
  ├────────────────────┼───────────┼────────────────────────────┤
  │ API Docs (MCPO)    │ 8000/docs │ MCPO Swagger UI            │
  └────────────────────┴───────────┴────────────────────────────┘

  5.8 문제 해결

  문제: sql 명령어를 찾을 수 없음

  # SQLcl 경로 확인
  which sql
  rpm -ql sqlcl

  # PATH에 없으면 추가
  echo 'export PATH=/opt/sqlcl/bin:$PATH' >> ~/.bashrc
  source ~/.bashrc

  문제: Java 버전 오류

  # Java 버전 확인
  java -version

  # 여러 Java가 설치된 경우 Oracle JDK 선택
  sudo alternatives --config java

  문제: Wallet 인증 실패

  # Wallet 파일 존재 확인
  ls -la /home/opc/wallet/

  # tnsnames.ora 확인
  cat /home/opc/wallet/tnsnames.ora

  # SQLcl로 직접 접속 테스트
  export TNS_ADMIN=/home/opc/wallet
  sql OPENWEBUIUSER@handson26ai_medium

  문제: MCPO 포트 충돌

  # 포트 사용 확인
  sudo ss -tlnp | grep 8000

  # 다른 포트로 실행
  uvx mcpo --port 8001 --api-key "your-secret-key" --config ./config.json


  5.9 SQLcl Named Connection 저장

  `list-connections` 엔드포인트에서 연결 목록이 조회되려면 SQLcl의 연결 관리자에 named connection을
  먼저 저장해야 합니다. MCPO 실행 전(또는 후)에 아래 명령을 한 번만 실행하면 영구 저장됩니다.

  #### 5.9.1 Named Connection 저장 방법

  ```bash
  # TNS_ADMIN을 설정한 뒤 SQLcl /nolog 모드에서 저장
  TNS_ADMIN=/home/opc/wallet sql /nolog <<'EOF'
  conn -save adwrent_medium -savepwd OPENWEBUIUSER/Oracle_12345@adwrent_medium
  conn -save adwrent_low    -savepwd OPENWEBUIUSER/Oracle_12345@adwrent_low
  conn -save adwrent_high   -savepwd OPENWEBUIUSER/Oracle_12345@adwrent_high
  exit
  EOF
  ```

  옵션 설명:
  - `-save <이름>` — 저장할 connection 이름 (list-connections에서 조회되는 이름)
  - `-savepwd`     — 비밀번호도 함께 저장 (없으면 연결 시 매번 입력 필요)

  #### 5.9.2 저장 결과 확인

  ```bash
  TNS_ADMIN=/home/opc/wallet sql /nolog <<'EOF'
  connmgr list -flat
  exit
  EOF
  # adwrent_high
  # adwrent_low
  # adwrent_medium
  ```

  #### 5.9.3 MCPO API로 확인

  ```bash
  curl -s -X POST http://localhost:8000/sqlcl/list-connections \
    -H "Authorization: Bearer mcposecret" \
    -H "Content-Type: application/json" \
    -d '{"model": "claude-sonnet-4-6", "definition_type": "all", "show_details": true}'
  ```

  #### 5.9.4 Named Connection으로 연결

  저장된 이름으로 `/connect` 엔드포인트를 호출하면 DB 버전, NLS 파라미터 등
  컨텍스트 정보를 자동으로 반환합니다.

  ```bash
  curl -s -X POST http://localhost:8000/sqlcl/connect \
    -H "Authorization: Bearer mcposecret" \
    -H "Content-Type: application/json" \
    -d '{"connection_name": "adwrent_medium", "model": "claude-sonnet-4-6"}'
  ```

  > **주의**: Named Connection은 SQLcl을 실행하는 OS 사용자(`opc`) 기준으로 저장됩니다.
  > Wallet이 교체된 경우 tnsnames.ora의 서비스 이름이 바뀔 수 있으므로,
  > 교체 후 connection을 다시 저장해야 합니다.

  #### 5.9.5 Connection 삭제

  ```bash
  TNS_ADMIN=/home/opc/wallet sql /nolog <<'EOF'
  connmgr delete -conn adwrent_medium
  exit
  EOF
  ```

---

OpenWebUI에서 MCPO를 연결하는 화면 입력 가이드

- Admin - Settings - External Tools 
                                                        ┌───────────────────────────┬────────────────────────────────────────────────┐
  │           필드            │                       값                       │             
  ├───────────────────────────┼────────────────────────────────────────────────┤             
  │ Type                      │ OpenAPI (이미 선택됨)                          │             
  ├───────────────────────────┼────────────────────────────────────────────────┤             
  │ URL                       │ http://158.179.172.126:8000 (이미 입력됨)      │
  ├───────────────────────────┼────────────────────────────────────────────────┤
  │ OpenAPI Spec              │ http://158.179.172.126:8000/sqlcl/openapi.json │
  ├───────────────────────────┼────────────────────────────────────────────────┤
  │ Auth                      │ Bearer / mcposecret (이미 입력됨)              │
  ├───────────────────────────┼────────────────────────────────────────────────┤
  │ Headers                   │ 비워두기                                       │
  ├───────────────────────────┼────────────────────────────────────────────────┤
  │ ID                        │ 비워두기                                       │
  ├───────────────────────────┼────────────────────────────────────────────────┤
  │ Name                      │ SQLcl MCP                                      │
  ├───────────────────────────┼────────────────────────────────────────────────┤
  │ Description               │ Oracle Database SQLcl MCP Server               │
  ├───────────────────────────┼────────────────────────────────────────────────┤
  │ Function Name Filter List │ 비워두기                                       │
  └───────────────────────────┴────────────────────────────────────────────────┘

                                                                          

