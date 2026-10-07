# G1 SONIC + PICO + Dex1 데이터 수집 실행 순서

업데이트: 2026-10-08. 실제 G1 + Dex1 양손, 헤드 스테레오 영상 한쪽과 양손목 D405,
LeRobot v2.1 형식의 30Hz 수집을 위한 로컬 실행 안내다.

각 번호는 별도 PC 터미널이다. SSH 명령 이후에는 해당 터미널이 Orin에 접속된 상태다.
이미 실행 중인 서비스는 중복 실행하지 않는다.

## 시작 전 확인

- 로봇 지지 장비와 비상정지를 준비한다. 힘 빠짐, 이상 진동, 걸림, 발열이 있으면 진행하지 않는다.
- SONIC 실행은 전신 제어다. 상체만 따라 움직이더라도 하체가 비활성이라는 뜻은 아니다.
- 기존 Dex1 개폐 보정 파일을 사용한다: `/mnt/storage/user/sonic_datasets/dex1_calibration.json`.
  장착이나 영점을 바꿨다면 기존 값이 맞는지 다시 확인한다. `--calibration`으로 모터 영점을 임의 변경하지 않는다.
- 사용자가 단일 손 개폐와 사과 접촉 후 차단을 확인했으나, 최신 PICO 차단/열림 복귀 로직은
  소프트웨어 테스트만 완료했다. 빈손에서 먼저 확인한다. Kp 5.0이 모든 물체에 안전하다는 뜻은 아니다.
- 손 차단은 위치 진행량 기반이며 힘/온도 제한이 아니다. 차단 시 물체가 떨어질 수 있다.
- 로봇이 서 있는 상태에서 제어기를 강제 종료하거나 전원을 갑자기 끄지 않는다.

## 1. PC: 네트워크와 포트 확인

```bash
ip -br addr show enp130s0
ping -c 2 192.168.123.161
ping -c 2 192.168.123.164
ss -lntp | grep -E ':5556|:5557|:5560'
```

기존 설정은 PC `192.168.123.222/24`, Orin `192.168.123.164`다.
ping 성공은 IP 통신 확인이며 DDS LowState 수신이나 모터 제어 준비까지 보장하지 않는다.
포트를 사용 중인 프로세스가 있으면 기존 실행 터미널을 확인한다. 무작정 종료하지 않는다.

## 2. PC: SONIC 실행

```bash
conda activate tel_rel_sonic
cd ~/Desktop/TELEOP_REAL/GR00T-WholeBodyControl/gear_sonic_deploy
./deploy.sh enp130s0 --input-type zmq_manager
```

실기 실행 확인 후 `Init Done`을 확인한다. `LowState is not available`가 반복되면
policy를 시작하지 말고 연결부터 확인한다. `Address already in use`는 중복 실행 여부를 확인한다.

## 3. PC: PICO 연결 서비스

```bash
pgrep -af RoboticsService
```

서비스가 없을 때만 다음을 실행한다.

```bash
bash /opt/apps/roboticsservice/runService.sh
```

PICO의 XRoboToolkit 앱을 열고 같은 네트워크의 워크스테이션에 연결한다.

## 4. PC: PICO manager

```bash
conda activate tel_rel_sonic
cd ~/Desktop/TELEOP_REAL/GR00T-WholeBodyControl
python gear_sonic/scripts/pico_manager_thread_server.py \
  --manager --vis_vr3pt --vis_smpl --record-controls right-stick
```

안전한 기준 자세에서 A+B+X+Y로 policy 시작/캘리브레이션을 수행하고 `PLANNER`를 확인한다.
왼쪽 스틱 클릭으로 `PLANNER_VR_3PT`를 선택한다. 이 과정에서 로봇이 움직일 수 있다.
버튼 조합은 시작/정지 토글이므로 이미 활성화된 상태에서 반복해서 누르지 않는다.
PICO 데이터나 로봇 상태가 없으면 다음 단계로 넘어가지 않는다.

## 5. PC에서 SSH: 카메라 송출

```bash
ssh -t unitree@192.168.123.164 \
 'env LD_PRELOAD=/home/unitree/miniforge3/envs/g1-vision/lib/libgomp.so.1 /home/unitree/miniforge3/envs/g1-vision/bin/python -u /home/unitree/sonic_collection/orin_collection_cameras.py --head-half first --head-rotate 180 --right-wrist-rotate 180'
```

- 헤드: 스테레오 영상의 첫 번째 절반, 180도 회전.
- 왼손목: D405 `260422275890`, 회전 없음.
- 오른손목: D405 `315122271725`, 180도 회전.

기존 송출기가 있다면 두 번째로 실행하지 않는다.

## 6. PC에서 SSH: 일반 Dex1 서버

```bash
ssh -t unitree@192.168.123.164
cd ~/dex1_1_service/bin
sudo ./dex1_1_gripper_server --network eth0
```

`Side: left`, `Side: right`, `Gripper Server started`를 확인하고 켜둔다.
일반 서버를 사용한다. `dex1_1_diagnostics`는 개폐 명령을 무시하며,
`dex1_1_left_trace`는 양손 텔레옵용 서버가 아니다. 서버들은 동시 실행하지 않는다.

## 7. PC: PICO 트리거로 Dex1 제어

기존 Dex1 어댑터와 단일 손 테스트를 종료하고, 물체를 내려놓고 손 주변을 비운 후 실행한다.
SONIC 몸체 제어나 PICO manager를 이 단계 때문에 종료하지 않는다.

```bash
cd ~/Desktop/TELEOP_REAL/GR00T-WholeBodyControl

~/miniconda3/envs/tel_rel_sonic/bin/python \
  gear_sonic/scripts/pico_dex1_adapter.py \
  --enable-write --kp 5.0 --kd 0.05 --max-speed 2.0
```

최대 목표 변경 속도는 2.0rad/s이며 실제 손 속도를 보장하지 않는다.
`feedback=True`와 최신 PICO 입력을 확인한다. 양쪽 트리거를 0.5초 이상 놓으면
`ARMED`가 출력되고 양손이 현재 위치에서 열린 위치로 움직인다.

### 트리거와 차단 동작

| 상태 | 동작 |
| --- | --- |
| `TRACKING` | 놓으면 열림, 당기면 닫힘. 중간 입력은 중간 목표 위치. |
| `BLOCKED` | 닫는 중 진행량 부족 감지. 해당 손만 추가 닫기 차단 및 Kp/Kd/토크 0 전송. 프로그램은 유지. |
| `OPENING` | 차단된 손의 트리거를 완전히 놓으면 열린 위치로 이동. 도중 다시 당겨도 닫히지 않음. |
| 복귀 | 열린 위치의 0.05rad 이내에서 트리거를 0.5초 놓으면 `TRACKING` 복귀. |

목표 선행은 약 0.10rad로 제한한다. 속도 2.0 설정에서는 위치 오차가 지속되는 동안
0.25초 진행량이 0.10rad 미만이면 닫힘을 차단한다. 정상적인 느린 움직임에도 차단될 수 있다.
접촉 감지나 과부하의 확정 진단은 아니다.

한 손의 닫힘 차단은 다른 손 제어를 종료하지 않는다. 단, 여는 중에도 막힘, 큰 추종 오차,
입력/상태 끊김, 범위 이탈, 제어 주기 100ms 초과는 전체 오류로 처리한다.
전체 오류나 종료 시 양손에 힘 해제 명령을 반복 전송하고 종료한다.
송신 성공은 모터 수신 확인이 아니며, 통신 단절/강제 종료 시 Orin timeout에 의존한다.

**차단된 손의 파지력은 유지하지 않는다. 전체 오류 시 양손 물체가 떨어질 수 있다.**
최신 복귀 동작을 빈손에서 검증한 뒤 녹화로 넘어간다. 셸 프롬프트로 돌아왔다면
단순 `BLOCKED`가 아니라 프로그램 종료이므로 마지막 오류를 확인하고 반복 구동하지 않는다.

## 8. PC: Dex1 녹화용 브리지

```bash
cd ~/Desktop/TELEOP_REAL/GR00T-WholeBodyControl
~/miniconda3/envs/tel_rel_sonic/bin/python \
  gear_sonic/scripts/bridge_dex1_recording.py --network-interface enp130s0
```

양손 상태와 양손 명령, 총 4개 채널 모두 수신되어야 한다.
이 브리지는 녹화용이며 모터 제어 명령을 발행하지 않는다.

## 9. PC: 전체 검사 후 녹화

```bash
cd ~/Desktop/TELEOP_REAL/GR00T-WholeBodyControl
bash gear_sonic/scripts/unifolm.sh check --seconds 10
```

검사가 통과한 뒤 같은 터미널에서 다음을 실행한다.
현재 태스크는 **사과를 초록색 접시로, 바나나를 하늘색 접시로 옮기기**다.

```bash
bash gear_sonic/scripts/unifolm.sh record \
  --dataset-name "apple_banana_sort_$(date +%Y%m%d_%H%M%S)" \
  --task "Place the apple on the green plate and the banana on the light blue plate."
```

다른 작업이면 `--task`를 실제 작업에 맞게 바꾼다. 녹화 중인 데이터셋의 태스크를 바꾸지 말고 새 수집기로 시작한다.

- 오른쪽 스틱 짧게 누르고 떼기: 에피소드 시작/저장.
- 오른쪽 스틱 1.5초 이상 누르고 떼기: 실패 처리.
- 먼저 5~10초만 녹화하고 저장 결과를 확인한다.
- 에피소드 저장 후 수집기에서 `q` + Enter로 종료한다.
- 수집기 READY 문구에 기존 Grip 조합이 나오더라도 이 문서의 manager는 `right-stick` 설정을 사용한다.

## 저장 위치 및 종료 주의

```text
/mnt/storage/user/sonic_datasets/apple_banana_sort_<날짜_시간>/
```

30Hz LeRobot v2.1 데이터셋에 Parquet, 카메라별 MP4, 메타데이터가 저장된다.
실제 유효 저장 여부는 수집기 로그와 검사 결과로 확인한다.
녹화 중 episode 파일이나 메타데이터를 삭제/이동하지 않는다. 이전에 이것으로 저장 개수 검사가 실패했다.
30Hz 지연이나 동기화 문제로 실패 처리되면 이를 무시하고 학습에 사용하지 않는다.

녹화 종료는 로봇 정지가 아니다. 먼저 물체를 안전하게 내려놓고, 로봇을 지지한 상태에서
정해진 실기 정지 절차를 따른다. Dex1 어댑터 종료는 파지력을 해제할 수 있다.
로봇이 몸체 제어에 의존해 서 있는 동안 SONIC을 강제 종료하지 않는다.

상세 내용: [Dex1 제어](DEX1_PICO_KO.md), [수집 포맷](UNIFOLM_COLLECTION_KO.md).
