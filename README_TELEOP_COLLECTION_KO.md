# SONIC부터 PICO 양손 제어와 녹화까지

업데이트: 2026-10-08.

기존 실행 순서를 유지하되 손 서버와 손 제어는 새 UnifoLM 구성을 사용한다.
녹화 브리지는 동일한 DDS 토픽을 사용하므로 기존 명령 그대로다.

각 번호는 **별도 PC 터미널**이다. SSH 이후 명령은 G1에서 실행된다.
이미 실행 중인 서비스는 중복 실행하지 않는다. G1 연결을 확인하고 로봇 지지 장비와
비상정지를 준비한다. SONIC은 전신 제어이며, 로봇이 서 있는 동안 강제 종료하지 않는다.

## 1. SONIC

PC에서 실행:

```bash
conda activate tel_rel_sonic
cd ~/Desktop/TELEOP_REAL/GR00T-WholeBodyControl/gear_sonic_deploy
./deploy.sh enp130s0 --input-type zmq_manager
```

`Init Done`을 확인한다. `LowState is not available`가 반복되면 다음으로 진행하지 않는다.

## 2. PICO 연결 서비스

PC에서 실행 여부 확인:

```bash
pgrep -af RoboticsService
```

서비스가 실행 중이지 않을 때만:

```bash
bash /opt/apps/roboticsservice/runService.sh
```

PICO의 XRoboToolkit 앱에서 워크스테이션에 연결한다.

## 3. PICO Manager

PC에서 실행:

```bash
conda activate tel_rel_sonic
cd ~/Desktop/TELEOP_REAL/GR00T-WholeBodyControl
python gear_sonic/scripts/pico_manager_thread_server.py \
  --manager --vis_vr3pt --vis_smpl --record-controls right-stick
```

기준 자세에서 **A+B+X+Y**로 policy 시작/캘리브레이션 후 `PLANNER`를 확인한다.
**왼쪽 스틱 클릭**으로 `PLANNER_VR_3PT`를 선택한다.
몸체가 움직일 수 있으며, 이미 활성화된 상태에서는 시작 조합을 다시 누르지 않는다.

## 4. 카메라

PC에서 SSH로 실행:

```bash
ssh -t unitree@192.168.123.164 \
 'env LD_PRELOAD=/home/unitree/miniforge3/envs/g1-vision/lib/libgomp.so.1 /home/unitree/miniforge3/envs/g1-vision/bin/python -u /home/unitree/sonic_collection/orin_collection_cameras.py --head-half first --head-rotate 180 --right-wrist-rotate 180'
```

헤드 첫 번째 절반은 180도 회전, 왼손목은 회전 없음, 오른손목은 180도 회전이다.
이미 카메라 송출기가 실행 중이면 생략한다.

## 5. 새 손 서버

이미 새 서버가 켜져 있으면 생략한다. 기존 손 서버와 동시에 실행하지 않는다.

```bash
ssh -t unitree@192.168.123.164
sudo bash ~/dex1_unifolm/start_server.sh
```

서버 터미널을 유지한다. `dex1_1_diagnostics`와 `dex1_1_left_trace`는 이 양손 서버의
대체 실행 명령이 아니다.

## 6. 새 PICO 손 제어

기존 `pico_dex1_adapter.py`와 수동 개폐·pulse 송신은 종료한 상태에서 진행한다.
SONIC과 PICO manager는 유지한다. **PC에서** 먼저 읽기 전용으로 확인:

```bash
bash ~/humanoid_challenge_ws/dex1_unifolm/run_pico.sh --seconds 5
```

`READY`와 양손 최신 피드백을 확인한 후 같은 터미널에서:

```bash
bash ~/humanoid_challenge_ws/dex1_unifolm/run_pico.sh --enable-write
```

양쪽 트리거를 놓고 있으면 `DDS READY` 후 새 입력과 0.5초 놓기를 확인해 `ARMED`가 된다.
왼쪽 트리거는 왼손, 오른쪽 트리거는 오른손이다. **당기면 닫힘, 놓으면 열림**이다.

공식 설정은 Kp 10, Kd 0.05, 200Hz, 측정 위치 기준 목표 제한 ±0.18rad다.
빈손에서 양손 개폐부터 확인한다. 접촉 시 진행량 부족으로 힘을 자동 해제하지 않으며,
힘/온도 제한은 아니다. 손 제어기의 입력·통신 끊김 처리는 유지된다.

## 7. 손 녹화 브리지

PC에서 실행:

```bash
cd ~/Desktop/TELEOP_REAL/GR00T-WholeBodyControl
~/miniconda3/envs/tel_rel_sonic/bin/python \
  gear_sonic/scripts/bridge_dex1_recording.py --network-interface enp130s0
```

양손 `state`·`cmd`, 총 4개 채널의 수신 개수가 올라오는지 확인한다.
이 브리지는 녹화용이며 모터 제어 명령을 발행하지 않는다.

## 8. 검사 후 녹화

PC에서 사전 검사:

```bash
cd ~/Desktop/TELEOP_REAL/GR00T-WholeBodyControl
bash gear_sonic/scripts/unifolm.sh check --seconds 30
```

`PREFLIGHT PASSED`를 확인한 뒤 같은 터미널에서 녹화 실행:

```bash
bash gear_sonic/scripts/unifolm.sh record \
  --dataset-name "apple_banana_sort_$(date +%Y%m%d_%H%M%S)" \
  --task "Place the apple on the green plate and the banana on the light blue plate."
```

사과를 초록색 접시로, 바나나를 하늘색 접시로 옮기는 태스크다.
다른 작업이면 `--task`를 바꾸고, 녹화 중인 데이터셋의 태스크는 변경하지 않는다.

- 오른쪽 스틱 짧게 눌렀다 떼기: 에피소드 시작/저장.
- 오른쪽 스틱 1.5초 이상 눌렀다 떼기: 실패 저장.
- 키보드 `c` + Enter: 시작/저장. `x` + Enter: 실패 저장.
- 첫 에피소드는 5~10초만 저장해 확인한다.
- 저장 후 수집기에서 `q` + Enter로 종료한다.

## 녹화 표시와 지연 처리

| 상태 | 창 표시 |
| --- | --- |
| 녹화 시작 | 빨간 원과 `recoding start` |
| 저장 중 | `saving...` |
| 파일 저장 완료 | 초록색 `save!!` |
| 실패 저장 완료 | 노란색 `dissave!!` |
| 저장 오류 | `ERROR` |

상태 창을 닫아도 녹화는 종료되지 않는다. 창이 필요 없으면 녹화 명령에
`--no-status-window`를 추가한다.

녹화에는 전체 시간 제한이 없다. 기본 `--timing-policy warn`에서는 입력 지연,
카메라 시각 차이, 반복 프레임, 처리 지연 때문에 자동 실패 저장하지 않는다.
지연된 입력이나 이전 영상이 재사용될 수 있으며, 버퍼의 과거 이력이 소실되면
남은 가장 오래된 샘플을 사용하고 원본 시각과 시각 차이를 기록한다.

품질 경고는 `meta/quality.jsonl`의 `degraded=true`와 프레임별
`meta/sync_warnings.jsonl`에 남는다. **`save!!`는 파일 저장 완료이지 학습 품질 통과가 아니다.**
실제 채널 데이터 부재, 디코딩/형식 오류, 디스크 오류까지 무시하지는 않는다.

사전 `check`는 기존 엄격 기준으로 진단한다. 녹화에도 예전 자동 실패 기준을 적용하려면
`--timing-policy strict`를 추가한다. 이 설정은 SONIC·PICO·손 제어 보호 설정과 별개다.

## 저장 위치와 재시작

```text
/mnt/storage/user/sonic_datasets/apple_banana_sort_<날짜_시간>/
  episode_000000/
    data/
    videos/
    meta/
  episode_000001/
    data/
    videos/
    meta/
```

각 에피소드 폴더에 Parquet, 카메라별 MP4, 메타데이터가 함께 들어간다.
각 폴더는 독립된 30Hz LeRobot v2.1 데이터셋이며 내부 episode index는 0이다.
수집 순번은 폴더 이름과 `quality.jsonl`의 `session_episode_index`로 구분한다.

수정 적용 시 현재 에피소드를 마치고 **수집기만** `q` + Enter로 종료한 뒤,
새 dataset 이름으로 녹화 명령을 다시 실행한다. SONIC·PICO·손 서버는 그대로 유지한다.
기존 데이터는 자동 이동/수정하지 않는다.

**녹화 종료는 로봇 정지가 아니다.** 먼저 물체를 안전하게 내려놓고 로봇을 지지한 상태에서
실기 정지 절차를 따른다. 손 제어기 종료 시 파지력이 해제될 수 있다.

상세 안내: [전체 수집 안내](README_TELEOP_COLLECTION_KO.md),
[수집 포맷과 변환](UNIFOLM_COLLECTION_KO.md).
