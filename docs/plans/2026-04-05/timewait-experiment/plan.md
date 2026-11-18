# 계획서 (Plan) — TCP TIME_WAIT 실험

> 작성일: 2026-04-05
> 요구사항: TIME_WAIT 폭탄/포트 고갈을 전용 서버에서 실험

---

## 1. 목표

TCP TIME_WAIT 소켓이 고부하 환경에서 실제로 포트 고갈을 일으키는지 검증하고,
Keep-Alive, tcp_tw_reuse, 포트 범위 설정이 이를 어떻게 완화하는지 데이터로 확인한다.

---

## 2. 접근 방식

### 2.1 전용 Go 서버
- 최소한의 Go net/http 서버 (응답 즉시 반환, sleep 없음)
- 빠른 응답으로 초당 커넥션 수를 극대화하여 TIME_WAIT를 빨리 쌓기 위함
- Docker 컨테이너 (2 CPU, 512MB)
- `--network=host` 모드: Docker NAT 우회하여 TIME_WAIT를 명확히 관찰

### 2.2 k6 시나리오 (158에서 실행)
- `noConnectionReuse: true`: 매 요청마다 새 TCP 연결 (TIME_WAIT 발생)
- `noConnectionReuse: false` (기본): Keep-Alive 비교군
- VU를 점진적으로 증가하며 포트 고갈 시점 관찰

### 2.3 모니터링
- k6 PC(158)에서 TIME_WAIT ���켓 수 실시간 추적
- 서버(114)에서도 TIME_WAIT 추적 (Connection: close 시 서버 측에도 쌓일 수 있음)
- k6 에러율 (connection refused / EADDRNOTAVAIL) 모니터링

---

## 3. 변경 대상 파일

| 파일 경로 | 변경 내용 | 신규/수정 |
|----------|----------|----------|
| `server/main.go` | 전용 Go 서버 (즉시 응답) | 신규 |
| `server/go.mod` | Go 모듈 | 신규 |
| `server/Dockerfile` | Docker 이미지 | 신규 |
| `docker-compose.yml` | 서버 + 모니터링 | 신규 |
| `k6/scenarios/timewait-nokeep.js` | Keep-Alive OFF (noConnectionReuse) | 신규 |
| `k6/scenarios/timewait-keepalive.js` | Keep-Alive ON (기본) | 신규 |
| `k6/scenarios/timewait-ramp.js` | VU 점진 증가로 포트 고갈점 탐색 | 신규 |
| `monitor/collect-timewait.sh` | TIME_WAIT 소켓 수 실시간 수집 | 신규 |
| `k6/run-all.sh` | 전체 실험 순차 실행 | 신규 |
| `report/hypotheses.md` | 가설 문서 | 신규 |
| `.gitignore` | docs/ 제외 | 신규 |

---

## 4. 참고 코드 스니펫

### 4.1 k6 noConnectionReuse 설정

```javascript
export const options = {
  noConnectionReuse: true,
  stages: [
    { duration: '30s', target: 200 },
    { duration: '60s', target: 200 },
    { duration: '30s', target: 0 },
  ],
};
```
- 출처: k6 공식 문서 (리서치 결과)

**코드 분석:**

**사용된 메서드/함수:**
| 설정 | 소속 | 역할 | 기본값 |
|------|------|------|--------|
| `noConnectionReuse` | k6 options | true면 매 요청마다 새 TCP 연결 생성 | false (Keep-Alive) |

**문법/언어 기능 설명:**
- `noConnectionReuse: true`: VU가 반복 실행할 때 이전 연결을 닫고 새 연결을 맺음. TCP FIN → TIME_WAIT 발생.
- 기본값 false일 때 k6는 HTTP/1.1 Keep-Alive로 연결을 재사용.

**동작 방식:**
- `noConnectionReuse: true` → 각 `http.get()` 호출 후 k6가 연결을 닫음 → k6 PC(클라이언트)에 TIME_WAIT 소켓 생성 → 60초 유지 → 포트 점유 → 28,232 포트 초과 시 새 연결 불가 → k6 에러 (EADDRNOTAVAIL)

### 4.2 Go 서버 — 즉시 응답

```go
func handler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.Write([]byte(`{"status":"ok"}`))
}

func main() {
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}
```
- 출처: Go stdlib net/http 패턴

**코드 분석:**

**사용된 메서드/함수:**
| 메서드/함수 | 소속 | 역할 | 반환값 |
|------------|------|------|--------|
| `w.Write([]byte)` | http.ResponseWriter | 응답 바디 직접 쓰기 | (int, error) |

**동작 방식:**
- sleep 없이 즉시 JSON 응답. 응답 시간 < 1ms이므로 k6가 초당 수천 연결 생성 가능.
- Go net/http는 기본적으로 Keep-Alive를 유지. 클라이언트가 `Connection: close`를 보내면 서버가 응답 후 연결을 닫음 → 서버 측 TIME_WAIT.
- k6 `noConnectionReuse: true`일 때는 k6가 먼저 FIN → k6 측 TIME_WAIT.

### 4.3 TIME_WAIT 모니터링 스크립트

```bash
while true; do
  count=$(ss -tan state time-wait | wc -l)
  echo "$(date +%H:%M:%S) TIME_WAIT: $count"
  sleep 1
done
```
- 출처: Linux ss 유틸리티

**코드 분석:**

**사용된 메서드/함수:**
| 명령 | 역할 | 비고 |
|------|------|------|
| `ss -tan state time-wait` | TIME_WAIT 상태 TCP 소켓만 필터링 | -t: TCP, -a: 모든 상태, -n: 숫자 표시 |
| `wc -l` | 줄 수 카운트 | 헤더 1줄 포함, 실제 수는 count-1 |

**동작 방식:**
- 1초마다 현재 TIME_WAIT 소켓 수를 출력. k6 실행 중 이 값이 28,000에 접근하면 포트 고갈 임박.
- `ss`는 `netstat`보다 빠름 (커널 netlink 직접 조회).

---

## 5. 트레이드오프

| 취하는 것 | 포기하는 것 | 이유 |
|----------|-----------|------|
| --network=host | Docker 네트워크 격리 | NAT 없이 TIME_WAIT를 호스트에서 직접 관찰 |
| 즉시 응답 서버 | 현실적 워크로드 | 초당 커넥션 수 극대화로 TIME_WAIT를 빨리 쌓기 위함 |
| k6 PC에서 TIME_WAIT 관찰 | 서버 측 관찰 | k6가 FIN initiator → TIME_WAIT는 k6 PC에 쌓임 |

---

## 6. 구현 순서

1. git init + .gitignore + README 업데이트
2. Go 전용 서버 + Dockerfile + docker-compose.yml
3. 가설 문서 (report/hypotheses.md)
4. k6 시나리오 3개 (nokeep, keepalive, ramp)
5. 모니터링 스크립트 (collect-timewait.sh)
6. run-all.sh
7. Docker 빌드 + 서버 기동 + 테스트 실행
8. 결과 수집 + report/results.md

---

## 승인 상태

- [ ] 사용자 검토 완료
- [ ] 구현 착수 승인
