개발자이시니 **Docker**와 **WireGuard**를 기반으로 가장 깔끔하고 관리가 쉬운(Web UI 포함) 방식으로 구축하는 PRD를 작성해 드립니다.

이 문서는 라즈베리 파이에 **'Wg-Easy' (WireGuard + Web UI)** 컨테이너를 올리는 것을 목표로 합니다.

---

# PRD: Raspberry Pi Home VPN Server (Project "HomeTunnel")

## 1. 개요 (Overview)

* **프로젝트명:** HomeTunnel
* **목적:** 유휴 자원인 라즈베리 파이를 활용하여, 외부에서도 안전하게 홈 네트워크에 접속하고 공용 Wi-Fi 보안 위협을 방지하는 개인 전용 VPN 서버 구축.
* **핵심 가치:** * **비용 절감:** 기존 하드웨어 활용으로 운영 비용 '0원'.
* **보안:** 공공 장소에서의 트래픽 암호화.
* **접근성:** 외부에서 집 내부망(NAS, 개발 서버 등) 직접 접속.



## 2. 시스템 아키텍처 (Architecture)

1. **Client:** 스마트폰(iOS/Android), 노트북(Mac/Windows) - WireGuard 앱 설치.
2. **Internet:** 공용망.
3. **EntryPoint:** 공유기 (DDNS + Port Forwarding).
4. **Server:** Raspberry Pi (Docker Container).
* **Service:** `ghcr.io/wg-easy/wg-easy` (WireGuard 프로토콜 + 관리자 Web UI).



## 3. 기술 스택 (Tech Stack)

* **Hardware:** Raspberry Pi (모델 3B+ 이상 권장, 유선 LAN 연결 권장).
* **OS:** Raspberry Pi OS Lite (64-bit 권장, Headless 설정).
* **Virtualization:** Docker & Docker Compose.
* **VPN Protocol:** WireGuard (UDP).
* **Management:** Web UI (Dashboard 제공).

## 4. 요구사항 명세 (Requirements)

### 4.1. 하드웨어 및 네트워크 요구사항

* **고정 내부 IP:** 라즈베리 파이는 공유기 상에서 고정 IP(예: `192.168.0.100`)를 할당받아야 한다.
* **포트 포워딩 (Port Forwarding):** 공유기 설정에서 VPN 포트(UDP `51820`)를 라즈베리 파이 IP로 포워딩해야 한다.
* **DDNS 설정:** 유동 IP 환경(일반 가정집)을 고려하여, 도메인(예: `myhome.duckdns.org`)으로 접속 가능해야 한다.

### 4.2. 소프트웨어 기능 요구사항

* **간편한 클라이언트 등록:** QR 코드를 생성하여 모바일 기기에서 즉시 연결할 수 있어야 한다.
* **접속 현황 모니터링:** 현재 연결된 클라이언트와 데이터 사용량을 웹 대시보드에서 확인할 수 있어야 한다.
* **광고 차단 (Optional):** DNS 레벨에서 광고 트래픽을 차단할 수 있어야 한다 (Pi-hole 연동 가능성 열어둠).

## 5. 구현 가이드 (Implementation Guide)

개발자이시므로 터미널 명령 위주로 핵심만 정리했습니다.

### Phase 1: 환경 설정 (Pre-requisites)

1. **OS 설치:** Raspberry Pi Imager로 'Raspberry Pi OS Lite' 설치 (SSH 활성화 필수).
2. **Docker 설치:**
```bash
curl -sSL https://get.docker.com | sh
sudo usermod -aG docker $USER

```



### Phase 2: docker-compose 작성 (Deployment)

홈 디렉토리에 `vpn` 폴더를 만들고 `docker-compose.yml`을 작성합니다.

```yaml
version: "3.8"
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy
    container_name: wg-easy
    environment:
      # ⚠️ 중요: 본인의 DDNS 주소로 변경 (예: myhome.iptime.org)
      - WG_HOST=YOUR_DDNS_DOMAIN 
      
      # Web UI 접속 비밀번호 (변경 필수)
      - PASSWORD=YOUR_ADMIN_PASSWORD
      
      # 기본 포트 설정
      - WG_PORT=51820
      - WG_DEFAULT_ADDRESS=10.8.0.x
      - WG_DEFAULT_DNS=1.1.1.1 # Cloudflare DNS
      
      # MTU 설정 (이슈 발생 시 조정, 보통 1420 or 1280)
      - WG_MTU=1420 
      
    volumes:
      - ./.wg-easy:/etc/wireguard
    ports:
      - "51820:51820/udp" # VPN 트래픽
      - "51821:51821/tcp" # Web UI 관리자 페이지
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped

```

### Phase 3: 실행 및 공유기 설정 (Execution)

1. **컨테이너 실행:**
```bash
docker compose up -d

```


2. **공유기 설정 (Router Admin Page):**
* **Port Forwarding:** 외부 포트 `51820` (UDP) → 내부 IP (라즈베리 파이) `51820` (UDP).
* **DDNS:** 공유기 제조사(iptime 등)에서 제공하는 DDNS 혹은 DuckDNS 설정.



### Phase 4: 클라이언트 연결 (Onboarding)

1. PC/모바일 브라우저에서 `http://[라즈베리파이IP]:51821` 접속.
2. 설정한 `PASSWORD`로 로그인.
3. `+ New Client` 버튼 클릭 후 이름(예: `MyPhone`) 입력.
4. 생성된 **QR 코드**를 스마트폰 WireGuard 앱으로 스캔하면 끝.

## 6. 위험 관리 및 유지보수 (Risks & Maintenance)

* **보안:** Web UI 포트(`51821`)는 공유기에서 포트 포워딩하지 **않는 것**을 권장합니다. (집 내부망에서만 관리자 페이지 접속).
* **속도:** 가정용 인터넷의 업로드 속도가 VPN 사용 시의 다운로드 속도가 됩니다. (비대칭 인터넷망 주의).
* **백업:** SD 카드는 수명이 있으므로, `./.wg-easy` 폴더(설정 파일)를 주기적으로 백업해야 합니다.

---

### 💡 개발자 팁 (Next Step)

이 구축이 완료되면, 집에 있는 **나스(NAS)나 개발용 서버**를 외부에서 공인 IP 노출 없이 `192.168.x.x` 내부 IP로 바로 접근할 수 있게 됩니다.

작업하시다가 `docker-compose` 설정이나 공유기 포트 포워딩 쪽에서 막히시면 말씀해주세요.