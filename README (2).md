# Phase 1 — WSL2에서 SICNav/CrowdSimPlus 연구 환경 구축하기

> **목적:** 처음 설치하는 연구자가 Windows + WSL2(Ubuntu) + Conda에서 SICNav의 CAMPC 시뮬레이션을 실행하고, 결과 저장 및 Phase 1 검증까지 재현할 수 있도록 안내합니다.  
> **검증 기준:** 2026-10-14, 기존 연구 환경의 Python 3.8.13 / CPU / `hallway_bottleneck` / 보행자 3명 / Test Case 799.  
> **중요:** 문서의 *실행 결과*는 기존 연구자의 검증 기록입니다. 아래 전체 설치 절차를 완전히 새로운 PC에서 처음부터 재검증한 것은 아닙니다. 운영체제·패키지 버전에 따라 추가 조정이 필요할 수 있습니다.

## 0. 빠른 길잡이

| 구간 | 진행할 작업 | 필수 여부 |
|---|---|---|
| 1~2 | WSL2 및 Ubuntu 기본 도구 설치 | 필수 |
| 3 | Miniconda + Python 3.8.13 환경 생성 | 필수 |
| 4~5 | SICNav 소스와 Python 의존성 설치 | 필수 |
| 6 | Python-RVO2 설치 | 필수 |
| 7 | CasADi / RVO2 / Gym 패키지 점검 | 필수 |
| 8 | 기본 CAMPC 실행·영상 확인 | 필수 |
| 9 | acados / HSL 설치 | **선택** |
| 10~12 | 연구용 로깅, CSV 변환, 재현성·Replay 검증 | **Phase 1 연구 확장** |

**기본 실행까지 필요한 부분은 1~8장입니다.** 현재 확인된 CAMPC 기본 경로는 **CasADi Opti + IPOPT**이며, **GPU와 acados 없이 CPU에서 실행**할 수 있습니다. acados는 실시간 SICNav 및 다른 정책을 위한 선택 구성입니다.

---

## 1. Windows에서 WSL2 + Ubuntu 준비

**Windows PowerShell을 관리자 권한으로 열고:**

```powershell
wsl --install -d Ubuntu
```

설치가 완료되면 Windows를 재시작하고 Ubuntu를 실행하여 Linux 계정 이름과 암호를 지정합니다. 기존에 WSL이 설치되어 있다면 다음으로 상태를 확인합니다.

```powershell
wsl --list --verbose
wsl --version
```

`Ubuntu`의 버전이 `2`인지 확인합니다. Ubuntu가 WSL1이라면, **실제 배포판 이름**을 확인한 뒤 다음 명령을 사용합니다.

```powershell
wsl --set-version Ubuntu 2
```

이후 모든 명령은 별도 표시가 없는 한 **Ubuntu(WSL) 터미널**에서 실행합니다. 저장소와 Conda는 속도·호환성을 위해 `/mnt/c/...`가 아닌 **WSL의 Linux 홈 디렉터리**에 설치하는 것을 권장합니다.

> 연구자가 성공적으로 사용한 플랫폼은 Windows의 WSL Ubuntu입니다. Ubuntu 버전·CPU 모델·메모리 용량은 현재 기록만으로 특정할 수 없으므로 절대적인 최소 사양을 주장하지 않습니다.

## 2. Ubuntu 필수 도구 설치

```bash
sudo apt update
sudo apt install -y \
  git build-essential cmake pkg-config \
  python3-dev python3-pip \
  ffmpeg curl wget unzip ca-certificates
```

버전을 확인합니다.

```bash
git --version
cmake --version
ffmpeg -version | head -n 1
```

- `build-essential`, `python3-dev`: Python-RVO2 등 C/C++ 확장 모듈 빌드용
- `cmake`: 선택 설치인 acados 빌드에도 사용
- `ffmpeg`: 시뮬레이션 영상을 MP4로 저장할 때 필요

## 3. Miniconda 설치 및 연구용 가상환경 생성

Conda가 이미 설치되어 있다면 **3.1을 생략**하고 3.2부터 진행합니다.

### 3.1 Miniconda 설치 (x86_64 Ubuntu 예시)

```bash
cd ~
uname -m
```

출력이 `x86_64`일 때 아래 설치 파일을 사용합니다. ARM(`aarch64`)이라면 해당 아키텍처의 공식 설치 파일을 선택해야 합니다.

```bash
wget -O miniconda.sh \
  https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash miniconda.sh
```

설치 안내에 따라 진행한 후 터미널을 다시 열거나 다음 명령을 실행합니다.

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda --version
```

`~/miniconda3`가 아니라 다른 위치를 선택했다면 해당 설치 경로로 바꿉니다.

### 3.2 Python 3.8.13 환경 생성

```bash
conda create -n sicnav_research python=3.8.13 -y
conda activate sicnav_research
python --version
python -m pip --version
```

기대 Python 버전:

```text
Python 3.8.13
```

> 기존 연구에서 실제 사용한 환경 이름은 `sicnav_research`입니다. 다른 이름도 가능하지만 이 문서는 동일한 이름을 기준으로 작성합니다.

## 4. 연구 디렉터리 생성 및 SICNav 소스 확보

```bash
mkdir -p ~/MPC_Crowd_Navigation_Research
cd ~/MPC_Crowd_Navigation_Research

git clone https://github.com/sepsamavi/safe-interactive-crowdnav.git
cd safe-interactive-crowdnav
```

기존 Phase 1의 소스 기준점은 다음 커밋입니다.

```bash
git fetch --all --tags
git checkout c702fb8ac9ba6439ca61da7dde68b8524bbc6a1f
git rev-parse HEAD
```

**왜 커밋을 고정하나요?** 공식 저장소의 최신 버전이 달라져도, 당시 분석·실험한 코드 경로를 최대한 재현하기 위함입니다. 이 커밋이 현재 원격 저장소에서 접근 가능한지는 체크아웃 성공 여부로 확인해야 합니다. 실패한다면 `git log --oneline`으로 이용 가능한 커밋을 확인하고, 사용한 해시를 실행 기록에 남기세요.

기본 파일을 확인합니다.

```bash
ls -l simple_test.py requirements.txt setup.py
ls -l sicnav/configs/env.config sicnav/configs/policy.config
```

**주의:** 설정 경로는 `sicnav/config/`가 아니라 **`sicnav/configs/`**입니다.

## 5. SICNav 및 Python 의존성 설치

아래 작업은 `sicnav_research` Conda 환경이 활성화된 상태에서 진행합니다.

```bash
cd ~/MPC_Crowd_Navigation_Research/safe-interactive-crowdnav
conda activate sicnav_research

python -m pip install -e .
python -m pip install -r requirements.txt
```

공식 SICNav README의 설치 순서를 따릅니다. 환경 구축 당시 원본 requirements에는 Python-RVO2 빌드와 호환되는 **Cython 0.29.33**가 안내되어 있습니다. 실제 설치 버전을 검사합니다.

```bash
python - <<'PY'
import sys
import Cython
print('Python:', sys.version.split()[0])
print('Cython:', Cython.__version__)
PY
```

만약 Python-RVO2 빌드에서 **Cython 3.x 관련 오류**가 발생하면 다음처럼 조정합니다.

```bash
python -m pip install 'Cython==0.29.33'
```

새 버전의 pip/빌드 도구가 Python 3.8을 지원하지 않는 경우에는 환경과 requirements를 먼저 확인하고, 무조건 최신 버전으로 업그레이드하지 마세요.

## 6. Python-RVO2 설치 (필수)

SICNav의 ORCA 기반 군중 시뮬레이션에는 Python-RVO2가 필요합니다. SICNav 내부 소스만 설치해서는 `import rvo2`가 정상 동작하지 않을 수 있습니다.

```bash
cd ~/MPC_Crowd_Navigation_Research

git clone https://github.com/sybrenstuvel/Python-RVO2.git
cd Python-RVO2

conda activate sicnav_research
python setup.py build
python setup.py install
python -m pip install -e .
```

> 공식 저장소의 설치 안내에 따른 절차입니다. GitHub의 최신 Python-RVO2가 바뀐 경우에는 빌드 결과와 Python 3.8 호환성을 따로 확인하세요.

설치 성공 여부 확인:

```bash
python - <<'PY'
import rvo2
print('RVO2 import: OK')
print('Simulator class:', rvo2.PyRVOSimulator)
PY
```

## 7. 필수 패키지 통합 점검

```bash
conda activate sicnav_research
cd ~/MPC_Crowd_Navigation_Research/safe-interactive-crowdnav

python - <<'PY'
import sys
import casadi
import rvo2
import crowd_sim_plus
print('Python:', sys.version.split()[0])
print('CasADi:', casadi.__version__)
print('RVO2: OK')
print('CrowdSimPlus: OK')
PY
```

모듈이 모두 정상적으로 import되면 기본 실행 준비가 완료된 것입니다.

의존성 기록도 저장해 둡니다.

```bash
mkdir -p ~/MPC_Crowd_Navigation_Research/research_results/phase01/setup_metadata
python -m pip freeze > ~/MPC_Crowd_Navigation_Research/research_results/phase01/setup_metadata/pip_freeze.txt
conda list --explicit > ~/MPC_Crowd_Navigation_Research/research_results/phase01/setup_metadata/conda_explicit.txt
git rev-parse HEAD > ~/MPC_Crowd_Navigation_Research/research_results/phase01/setup_metadata/sicnav_commit.txt
```

## 8. 기본 CAMPC 시뮬레이션 실행

### 8.1 기준 Test Case 799 실행

```bash
conda activate sicnav_research
cd ~/MPC_Crowd_Navigation_Research/safe-interactive-crowdnav

python simple_test.py \
  --policy campc \
  --env_config sicnav/configs/env.config \
  --policy_config sicnav/configs/policy.config \
  --test_case 799
```

이 설정에서 확인된 대표 조건은 다음과 같습니다.

| 구성 | 당시 실험 기준 |
|---|---|
| 시나리오 | `hallway_bottleneck` |
| 보행자 | 3명 |
| 로봇 정책 | `campc` |
| 최적화 | CasADi Opti / IPOPT |
| MPC Horizon | 4 |
| 시뮬레이션 시간 간격 | 0.25초 |
| 실행 장치 | CPU |
| Test Case | 799 |

이 구성은 사용한 커밋과 설정 파일을 기준으로 합니다. 설치 환경이 다른 경우 출력값을 확인하세요.

### 8.2 성공 여부 확인

로그에 다음과 유사한 메시지가 나타나는지 확인합니다.

```text
[TEST PHASE] About to start test case 799.
[CrowdSimPlus] Success saving video via ffmpeg to: ...mp4
```

기존 연구자의 **기준 결과**는 다음과 같았습니다.

| 지표 | 기준 결과 |
|---|---:|
| 목표 도달 | 성공 (`test_case_success=1`) |
| 전체 Step | 19 |
| 시뮬레이션 시간 | 4.75초 |
| 충돌 | **1회** |
| 기존 환경의 정지 이벤트 수 | 0 |

**해석 주의:** 목표 도달 성공과 충돌 없는 성공은 다릅니다. `num_frozen=0`도 후속 연구의 '2초 이상 지속 정지'가 없었다는 뜻은 아닙니다.

기본 PKL/MP4는 일반적으로 `results_sicnav-np/` 아래 시나리오별 폴더에 생성됩니다.

```bash
find results_sicnav-np -type f \( -name '*tc_799.pkl' -o -name '*tc_799.mp4' \) -print
```

#### WSL의 영상 출력

영상 **저장**은 FFmpeg가 가능해야 합니다. GUI 창이 열리지 않아도 MP4 저장이 성공할 수 있습니다. 반면 실행 코드가 `plt.show()` 등 그래픽 창을 요구하는 경우에는 WSLg/GUI 지원 여부가 영향을 줄 수 있습니다.

---

## 9. 선택 설치: acados와 HSL

**CAMPC 기본 실행만 필요하면 이 장은 건너뛰어도 됩니다.** 기존 연구자는 acados를 별도로 설치했으나, 위 `--policy campc` 실행에서 사용하는 Solver는 CasADi/IPOPT로 확인됐습니다.

### 9.1 acados 소스 및 Python 바인딩

SICNav 공식 문서는 실시간 버전에 acados **v0.2.6에 대응하는 커밋 `285d382`**을 안내합니다.

```bash
cd ~/MPC_Crowd_Navigation_Research

git clone https://github.com/acados/acados.git
cd acados
git checkout 285d382
git submodule update --recursive --init

mkdir -p build
cd build
cmake -DACADOS_PYTHON=ON -DACADOS_WITH_QPOASES=ON ..
make -j2
make install
```

> 빌드 옵션과 `make install` 동작은 acados 버전 및 플랫폼에 따라 다를 수 있습니다. 오류가 발생하면 고정한 커밋의 설치 문서를 우선 확인하세요. 노트북 환경에서는 `make -j2`처럼 병렬성을 낮추어 빌드할 수 있습니다.

```bash
conda activate sicnav_research
python -m pip install -e ~/MPC_Crowd_Navigation_Research/acados/interfaces/acados_template

export ACADOS_SOURCE_DIR="$HOME/MPC_Crowd_Navigation_Research/acados"
export LD_LIBRARY_PATH="$ACADOS_SOURCE_DIR/lib:${LD_LIBRARY_PATH:-}"

python -c 'import acados_template; print("acados_template: OK")'
ls "$ACADOS_SOURCE_DIR/lib/libacados.so"
```

환경변수는 **현재 터미널에만 적용**됩니다. 필요하다면 `~/.bashrc`에 추가합니다(해당 컴퓨터에 acados가 설치된 경우에만).

```bash
cat >> ~/.bashrc <<'EOF'
export ACADOS_SOURCE_DIR="$HOME/MPC_Crowd_Navigation_Research/acados"
export LD_LIBRARY_PATH="$ACADOS_SOURCE_DIR/lib:${LD_LIBRARY_PATH:-}"
EOF
```

### 9.2 IPOPT/HSL

공식 SICNav 문서는 HSL 선형 Solver 설치를 *권장*하지만, 기본 IPOPT 설정으로도 실행 가능하다고 설명합니다. 본 연구에서도 기본 CAMPC 실행이 확인됐습니다. 따라서 **HSL 미설치를 치명적 오류로 취급하지 마세요.** 설치 시 [CasADi의 HSL 안내](https://github.com/casadi/casadi/wiki/Obtaining-HSL)를 참조합니다.

---

## 10. Phase 1 연구 확장: Step별 상태 및 Solver 로깅

여기서부터는 **공식 SICNav에 기본 포함된 기능이 아닌, 연구자가 추가한 코드**입니다. 새 환경에서 로깅·Replay를 그대로 재현하려면 수정된 `simple_test.py`와 `research_tools`를 함께 버전 관리하거나 복사해야 합니다.

### 10.1 원본 파일 보호

```bash
cd ~/MPC_Crowd_Navigation_Research/safe-interactive-crowdnav
cp simple_test.py simple_test.py.before_phase01_logging.bak
```

### 10.2 기본 PKL에 세 항목 추가

`simple_test.py`의 `summ_dict = { ... }` 블록(최종 `pickle.dump()` 직전)을 찾아 다음 세 필드를 **해당 딕셔너리 안에** 추가합니다.

```python
'recorded_states': deepcopy(states),
'recorded_actions': deepcopy(actions_array),
'per_step_summary': deepcopy(per_step_summary),
```

해당 스크립트는 이미 반복문에서 `states`, `actions_array`, `per_step_summary`를 수집하므로 기본적으로 새 객체를 만들 필요가 없습니다. 반복문의 기존 구조를 변경하지 마세요.

```bash
python -m py_compile simple_test.py
```

> 복사할 때 들여쓰기와 쉼표를 확인하세요. Python 3.8에서 문법 검사를 반드시 통과해야 합니다.

### 10.3 로깅 포함 실험 재실행

```bash
python simple_test.py \
  --policy campc \
  --env_config sicnav/configs/env.config \
  --policy_config sicnav/configs/policy.config \
  --test_case 799
```

**중요:** 동일 이름의 PKL/MP4가 덮어써질 수 있습니다. 기존 파일을 보존해야 한다면 실행 전 백업하거나 별도의 Test Case/파일명으로 구분하세요.

### 10.4 PKL 필드와 길이 검사

```bash
python - <<'PY'
import glob, pickle
files = sorted(glob.glob('results_sicnav-np/**/*tc_799.pkl', recursive=True))
if len(files) != 1:
    raise SystemExit('PKL 후보 수가 1개가 아닙니다: {}'.format(files))
with open(files[0], 'rb') as f:
    d = pickle.load(f)
n = d['num_steps']
solver = d['campc/campc_data']['solver_summary']
assert len(d['recorded_states']) == n + 1
assert len(d['recorded_actions']) == n
assert all(len(v) == n + 1 for v in d['per_step_summary'].values())
assert all(len(v) == n for v in solver.values())
print('LOGGING CHECK PASS:', 'steps=', n, 'states=', n + 1)
PY
```

`pickle.load()`는 **본인이 생성했거나 신뢰할 수 있는 파일만** 대상으로 실행해야 합니다.

### 10.5 분석용 CSV/JSON 변환

기존 연구자는 다음 보조 스크립트를 작성해 사용했습니다.

```text
research_tools/export_episode.py
```

이 파일은 **공식 저장소 설치만으로 생성되지 않습니다.** Phase 1 연구 코드가 있다면 해당 파일을 `research_tools/`에 복사한 다음 사용합니다.

```bash
mkdir -p research_tools
# 이 위치에 연구용 export_episode.py를 복사한 후:
PKL=$(find results_sicnav-np -name '*tc_799.pkl' -print -quit)
python research_tools/export_episode.py \
  --input "$PKL" \
  --output ~/MPC_Crowd_Navigation_Research/research_results/phase01/tc_799
```

당시 정상적으로 생성된 결과는 다음과 같습니다.

```text
research_results/phase01/tc_799/
├── robot_states.csv          # 20 rows + header
├── pedestrian_states.csv     # 60 rows + header
├── actions.csv               # 19 rows + header
├── mpc_diagnostics.csv       # 19 rows + header
├── environment_events.csv    # 20 rows + header
└── episode_summary.json
```

```bash
wc -l ~/MPC_Crowd_Navigation_Research/research_results/phase01/tc_799/*.csv
```

> 현재 `actions.csv`에는 **MPC가 반환한 명령**만 기록됩니다. 시뮬레이터가 안전 제약을 적용한 뒤의 실제 명령은 별도 계측이 필요합니다.

## 11. 동일 Test Case 3회 반복: 재현성 검증

연구 코드의 `recorded_states`, `recorded_actions` 저장 기능이 있어야 전체 궤적을 비교할 수 있습니다.

```bash
cd ~/MPC_Crowd_Navigation_Research/safe-interactive-crowdnav
OUT="$HOME/MPC_Crowd_Navigation_Research/research_results/phase01/reproducibility"
mkdir -p "$OUT"

for run in 1 2 3; do
  echo "=== RUN $run ==="
  if ! python simple_test.py \
      --policy campc \
      --env_config sicnav/configs/env.config \
      --policy_config sicnav/configs/policy.config \
      --test_case 799 > "$OUT/run_${run}.log" 2>&1; then
    echo "RUN $run failed; see $OUT/run_${run}.log"
    exit 1
  fi
  PKL=$(find results_sicnav-np -name '*tc_799.pkl' -print -quit)
  test -n "$PKL" || { echo 'PKL not found'; exit 1; }
  cp "$PKL" "$OUT/run_${run}.pkl"
done
```

검증 코드:

```bash
cd ~/MPC_Crowd_Navigation_Research/research_results/phase01/reproducibility
python - <<'PY'
import pickle
import numpy as np
runs = []
for i in (1, 2, 3):
    with open('run_{}.pkl'.format(i), 'rb') as f:
        runs.append(pickle.load(f))

def to_arrays(d):
    robot = np.asarray([[s.self_state.px, s.self_state.py] for s in d['recorded_states']])
    humans = np.asarray([[[h.px, h.py] for h in s.human_states] for s in d['recorded_states']])
    actions = np.asarray([[a.v, a.r] for a in d['recorded_actions']])
    return robot, humans, actions

for i in (1, 2):
    a, b = to_arrays(runs[0]), to_arrays(runs[i])
    print('RUN 1 vs', i + 1)
    for label, x, y in zip(('robot', 'humans', 'actions'), a, b):
        print(' ', label, 'identical:', np.array_equal(x, y))
    sa = runs[0]['campc/campc_data']['solver_summary']
    sb = runs[i]['campc/campc_data']['solver_summary']
    for key in ('optim_status', 'sol_success', 'iter_count', 'debug_text'):
        print(' ', key, 'identical:', list(sa[key]) == list(sb[key]))
PY
```

**기존 검증 결과:** 3회 모두 19 Step, 목표 도달, 충돌 1회, Freezing 이벤트 0회. 로봇·보행자 궤적과 제어 명령의 최대 차이 0, 주요 Solver 기록 완전 일치.

## 12. Step 8 Checkpoint → Restore → Replay (연구용 추가 모듈)

기존 연구에서 성공한 중간 시점 복원 검증은 다음 파일을 사용했습니다.

```text
research_tools/checkpoint.py
replay_test_phase01.py
```

두 파일은 **upstream 공식 SICNav 저장소에는 존재하지 않는 연구용 파일**입니다. 기존 연구자의 최종 백업은 다음 위치에 있습니다.

```text
~/MPC_Crowd_Navigation_Research/research_results/phase01/final_snapshot/
```

새 PC에서 복제하려면 **연구자가 별도로 공유한 해당 파일**을 사용해야 합니다. 로컬 백업을 재사용하는 경우:

```bash
cd ~/MPC_Crowd_Navigation_Research/safe-interactive-crowdnav
SNAP="$HOME/MPC_Crowd_Navigation_Research/research_results/phase01/final_snapshot"
cp "$SNAP/checkpoint.py" research_tools/checkpoint.py
cp "$SNAP/replay_test_phase01.py" ./replay_test_phase01.py
python -m py_compile research_tools/checkpoint.py replay_test_phase01.py
```

다음은 **기존 연구 코드가 준비되어 있을 때**만 실행할 수 있습니다.

```bash
python replay_test_phase01.py \
  --policy campc \
  --env_config sicnav/configs/env.config \
  --policy_config sicnav/configs/policy.config \
  --test_case 799 > phase01_replay_test.log 2>&1

echo "Exit code: $?"
grep -E '\[REPLAY\]|Traceback|AssertionError' phase01_replay_test.log | tail -n 30
```

기존 환경에서 확인된 결과:

```text
[REPLAY] Checkpoint captured at step 8
[REPLAY] Restoring checkpoint
[REPLAY] Baseline remaining steps: 11
[REPLAY] Replay remaining steps: 11
[REPLAY] robot max_abs_error: 0
[REPLAY] humans max_abs_error: 0
[REPLAY] action max_abs_error: 0
[REPLAY] optim_status identical: True
[REPLAY] sol_success identical: True
[REPLAY] iter_count identical: True
[REPLAY] debug_text identical: True
[REPLAY] PASS: checkpoint restoration verified
```

**검증된 범위:** 같은 Python 프로세스, 같은 정책, 같은 Test Case, Step 8 이후 11 Step. **아직 검증하지 않은 범위:** 별도 프로세스·파일에서 재시작, 대안 행동 적용, 모든 종류의 에피소드에서의 복원 성공.

---

## 13. 흔한 오류와 해결 방법

| 증상 | 원인/확인 항목 | 조치 |
|---|---|---|
| `conda: command not found` | Conda 셸 초기화가 적용되지 않음 | `source ~/miniconda3/etc/profile.d/conda.sh` 후 다시 실행 |
| `No module named rvo2` | Python-RVO2 빌드 또는 설치 누락 | 6장 설치 후 활성 Conda 환경에서 `import rvo2` 검사 |
| `No module named casadi` | Python 의존성 설치 실패/다른 환경 사용 | `which python`, `conda info --envs`, `pip install -r requirements.txt` 확인 |
| `sicnav/config/env.config` 파일 없음 | 설정 디렉터리 오타 | `sicnav/configs/env.config` 사용 |
| IPOPT `MA57`/HSL 경고 | 선택 HSL Solver가 없음 | 기본 IPOPT로 실행 가능한지 먼저 확인; 필요시에만 HSL 설치 |
| IPOPT `Maximum_Iterations_Exceeded` | 최적화 최대 반복 횟수 도달 | 즉시 실행 실패로 단정하지 말고 `sol_success`, 실제 행동 및 Solver 기록 함께 확인 |
| 영상이 저장되지 않음 | FFmpeg 부재 또는 그래픽 설정 | `which ffmpeg`, 2장 확인; 로그에서 실제 저장 실패 여부 확인 |
| 파일 `*.pkl`이 없음 | Test Case 경로가 다르거나 실행 실패 | `find results_sicnav-np -name '*.pkl'` 확인 |
| `recorded_states` KeyError | 원본 `simple_test.py`에는 추가 저장 필드가 없음 | 10장 수정 후 새로 실행 |
| `checkpoint.py` 없음 | 공식 저장소가 아닌 연구용 추가 코드 | 백업/연구 저장소에서 복사 |
| `acados_template` import 실패 | 선택 설치·라이브러리 경로 문제 | 9장 실행/`ACADOS_SOURCE_DIR`, `LD_LIBRARY_PATH` 확인 |
| WSL에서 GUI가 나타나지 않음 | WSLg 또는 디스플레이 설정 | 먼저 MP4 파일 생성 여부 확인. 필요할 때만 GUI 설정 점검 |

## 14. 최소 설치 완료 체크리스트

- [ ] Windows에서 WSL2 Ubuntu 실행 가능
- [ ] Conda `sicnav_research` 활성화 가능
- [ ] Python `3.8.13`
- [ ] SICNav 소스 다운로드 및 커밋 확인
- [ ] `pip install -e .` / `pip install -r requirements.txt` 성공
- [ ] `import casadi`, `import rvo2`, `import crowd_sim_plus` 성공
- [ ] `simple_test.py --policy campc ... --test_case 799` 실행 성공
- [ ] PKL/MP4 출력 확인
- [ ] (연구 확장) Step별 로깅 포함 PKL 및 CSV 변환 확인
- [ ] (연구 확장) 동일 Test Case 반복 재현성 확인
- [ ] (연구 확장) Snapshot → Restore → Replay 통과 확인

## 15. 폴더 구조 예시

```text
~/MPC_Crowd_Navigation_Research/
├── safe-interactive-crowdnav/       # SICNav upstream 소스 + 연구 확장
│   ├── simple_test.py
│   ├── requirements.txt
│   ├── sicnav/configs/
│   ├── research_tools/              # 연구자 추가 파일; upstream에는 없음
│   │   ├── export_episode.py
│   │   └── checkpoint.py
│   ├── replay_test_phase01.py      # 연구자 추가 Replay 검증
│   └── results_sicnav-np/          # 실행 후 결과
├── Python-RVO2/                   # 별도 클론·빌드
├── acados/                        # 선택 설치
└── research_results/phase01/
    ├── setup_metadata/
    ├── tc_799/
    ├── reproducibility/
    └── final_snapshot/
```

## 16. 참고 자료 및 재현성 원칙

- [공식 SICNav/SICNav-Diffusion 저장소](https://github.com/sepsamavi/safe-interactive-crowdnav)
- [Python-RVO2 저장소](https://github.com/sybrenstuvel/Python-RVO2)
- [acados 설치 안내](https://docs.acados.org/installation/index.html)
- [acados Python 인터페이스](https://docs.acados.org/python_interface/)
- [CasADi HSL 안내](https://github.com/casadi/casadi/wiki/Obtaining-HSL)

**재현성 원칙:** 실험 기록마다 **SICNav Git 커밋, Python 패키지 버전, Conda 환경, 시나리오 설정, Test Case, 연구용 코드 버전**을 보존하세요. 기존 Phase 1에서 확인된 재현성은 *동일 환경과 동일 정책* 기준이며, 다른 환경에서 동일 결과가 나오는지 여부는 별도 검증이 필요합니다.
