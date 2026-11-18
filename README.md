# tcp-timewait-lab

TCP TIME_WAIT 소켓 폭탄과 포트 고갈이 고부하 환경에서 실제로 발생하는지 검증한다.

## 배경

짧은 HTTP 요청을 대량으로 보내면 클라이언트 측에 `TIME_WAIT` 상태 소켓이 쌓인다.

- TIME_WAIT 기본 지속: 60초 (2 × MSL)
- 사용 가능 포트: ~28,000 (32768-60999, `net.ipv4.ip_local_port_range`)
- **28,000 / 60초 = 초당 ~467 연결이 이론적 한계**
- Keep-Alive를 쓰면 이 문제가 완화되지만, 커넥션 풀 미사용 시 발생

## 가설

| # | 가설 | 근거 |
|---|------|------|
| H1 | Keep-Alive OFF + 500 req/s에서 30초 내 포트 고갈 발생 | 500 × 60 = 30,000 TIME_WAIT |
| H2 | `tcp_tw_reuse=1` 설정 시 포트 고갈이 해소될 것 | TIME_WAIT 소켓 재사용 |
| H3 | Keep-Alive ON에서는 TIME_WAIT가 거의 발생하지 않을 것 | 커넥션 재사용 |
| H4 | sync-async-lab 고부하 테스트에서 실제로 TIME_WAIT가 발생했을 것 | 1000 VU에서 3000+ req/s |

## 실험 설계

### 시나리오
1. Keep-Alive OFF + 고부하 → TIME_WAIT 누적 관찰
2. Keep-Alive ON + 고부하 → TIME_WAIT 비교
3. `tcp_tw_reuse` ON/OFF 비교
4. `ip_local_port_range` 축소하여 포트 고갈 재현

### 측정
- `ss -s` 또는 `ss -tan state time-wait | wc -l`로 TIME_WAIT 소켓 수 추적
- k6 에러율 (connection refused) 모니터링
- `conntrack -S`로 커넥션 추적 테이블 상태

### sync-async-lab 사후 분석
- 이전 실험 환경에서 실제로 TIME_WAIT가 쌓였는지 확인
- Docker 환경에서의 네트워크 네임스페이스 영향
