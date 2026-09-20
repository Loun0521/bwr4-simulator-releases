# BWR-4 시뮬레이터 — 물리 레퍼런스 (physics.md)

> 2026-09-13 정정: APRM·RBM은 중성자 출력, 열수지는 붕괴열을 포함한 총 열출력을 사용한다.
> 24V 계측 배터리는 4시간이며 125V 가정과 별개다. 보이드·CPR은 채널별 질량속도를 사용한다.
> C01~C06·V01~V04 및 추가 검토 결과: [수정·UI 보고서](tests/BUGFIX_UI_REPORT.md).

이 문서는 **각 계통의 상태량이 코드에서 어떻게 계산되는지**를 변수 단위로
설명한다. `CHANGES.md`가 "무엇을 왜 이렇게 만들었나"를 서술한다면, 이
문서는 "이 값이 어느 수식에서, 어떤 상수로부터 나오는가"를 코드에 있는 그대로
정리한 참조용 문서다.

**용도**
1. 새 대화(채팅)를 시작할 때 이 문서를 먼저 읽으면 전체 물리를 다시 설명할
   필요가 없다.
2. 오류가 생겼을 때 코드 주석을 일일이 뒤지지 않고, 문제가 된 상태량이
   어느 계산 경로에서 나오는지 이 문서에서 바로 찾을 수 있다.

**파일 배치**
```
core/    원전 물리   bwr4_core.py  bwr4_bop.py     ← 이 문서가 다루는 대상
panel/   제어반      bwr_panel.py  bwr_diagram.py
tests/   검증        validate.py  subsys_test.py  ...
```

**표기 약속**
- 코드의 실제 변수명·상수명을 그대로 쓴다 (`self.core_power`, `VOID_COEF` 등).
- 단위: 시간 s, 온도 ℃, 압력 Pa(원자로 쪽)·kPa(2차계통 쪽), 질량 kg,
  에너지 J, 출력은 특별한 말이 없으면 정격 대비 분율(0~1).
- 상수값은 모두 코드에서 뽑은 실제 값이다. 코드가 바뀌면 이 문서도 그 자리에서 고친다.
- "정격"은 100% 출력 정상 운전점을 뜻한다.

**정격 기준값** (`RATED_*`)
| 상수 | 값 | 의미 |
|---|---|---|
| `RATED_MW` | 3293 MW | 정격 열출력 |
| `RATED_PRESSURE` | 7.03 MPa | 정격 원자로 압력 |
| `RATED_CORE_FLOW` | 13100 kg/s | 정격 노심 유량 |
| `RATED_STEAM_FLOW` | 1827 kg/s | 정격 증기 유량 |
| 발전기 | 약 1100 MWe | 총출력 (효율 약 33%) |

---

## 0. 계산 한 스텝의 순서 (`Reactor.step`)

한 스텝 `dt`마다 아래 순서로 각 계통을 푼다. 순서 자체가 물리적으로 의미가
있다 — 앞 계통의 결과가 뒷 계통의 입력이 된다.

```
step(dt):
  0. dt 세분화 (아래 '스텝 크기 자동 제어' 참조)
  1. _protection(dt)        보호계통 — 스크램 판정, 스크램 시 제어봉 삽입
  2. _kinetics(dt)          중성자 — k_eff, 출력 형상, 점동특성
     (주기 계산)          core_period 갱신
  3. _crd(dt)               제어봉 구동수압 — 축압기 충전·배수용기 수위
  4. _nms(dt)               기동영역 계측 — SRM 기간·IRM 레인지·검출기 위치
  5. _ac_power(dt)          소내 교류전원 — 상실 시 재순환 지령 삭제
     (bop._air)           계장공기는 _plant 안 bop.step 첫머리에서 푼다
  6. _diesel(dt)            비상 디젤발전기 — 기동·부하·연료·엔진냉각
  7. _rbsw(dt)              원자로건물 서비스수 — 최종 열침원
  8. _battery(dt)           125V 직류 4 계열 — 충전·방전
     _nms_power(dt)        24V 중성자계측 전원
     _rps_power(dt)        RPS 급전
     _tunnel(dt)           주증기 터널 열수지
  9. _isolation(dt)         주증기격리 — MSIV 여닫음
 10. _recirc_trip(dt)       재순환 펌프 트립·런백 신호
     _recirc(dt)            실제 속도 갱신 — 정상 추종 / 런백 / 정지 관성
 11. _slc(dt)               붕산주입 — 붕소 질량·노심 농도
 12. _rhr(dt)               잔열제거 — 수조냉각·격납살수
 13. _purge(dt)             격납용기 퍼지 (질소 배기)
 14. _containment(dt)       격납용기 열·압력
 15. _decay_heat(dt)        붕괴열 6군
 16. _fuel_temperature(dt)  연료·피복재 온도 (노심 노출 포함)
 17. _vessel(dt)            압력용기 — 급수·비등·압력·SRV·수위 (내부 10분할)
 18. _plant(dt)             2차계통 (bop.step)
 19. _channel_hydraulics(dt) 채널 열수력 — 보이드·건도·냉각재온도
 20. _thermal_limits(dt)    열적 제한 — MCPR·LHGR·APLHGR (5초마다)
 21. _eccs(dt)              비상노심냉각 — 신호·주입·수원·ADS·정지냉각
 22. _xenon(dt)             제논·아이오딘
 23. _samarium(dt)          사마륨·프로메튬
 24. _burnup(dt)            연소도
     (승온율 계산)        heatup_rate 갱신
```

> 이 목록은 `tests/conflict_test.py` 가 `step()` 원문에서 뽑아 쓰는 것과 같은
> 순서다. 한 스텝 안에서 **한 값에는 주인이 하나**여야 한다 — 둘이 서로 반대로
> 당기면 값이 어느 목표도 아닌 곳에 멈춘다 (§5.37 의 결함, VERIFICATION §13.15).

20번은 **되먹임이 없는 감시량**이라 매 스텝 풀지 않고 `THERMAL_UPDATE_DT`(5초)
간격으로만 갱신한다. 19번의 건도 프로파일을 그대로 입력으로 쓰므로 그 뒤에 온다.
5초 표본은 짧은 과도의 최저점을 놓칠 수 있다. 사고 관측에서는 필요한 간격으로
`thermal_limits()`를 직접 호출한다. 냉각 저하 되먹임은 매 물리 스텝 별도 판정한다.

### 스텝 크기 자동 제어

`step(dt, max_dt)` 는 `dt`가 너무 크면 스스로 잘게 쪼갠다. 상한 `cap`은:

- 기본 `MAX_DT` = 1.0 s
- `max_dt`를 주면 `MAX_DT_COARSE` = 20.0 s 로 잘라 그 이하만 허용
- 사고 징후(스크램·MSIV 폐쇄·SRV 개방·ECCS 유량>1)면 `ACCIDENT_MAX_DT` = 2.0 s
- 저압 대량 주입(ECCS 유량>50)이면 `INJECTION_MAX_DT` = 0.5 s
- 전원 상실 직후(펌프 코스트다운 중)면 `INJECTION_MAX_DT` = 0.5 s

이유: 이산 사건(밸브 개폐·펌프 트립)이나 급격한 응축은 스텝이 거칠면
결과가 흔들리기 때문이다. `dt > cap`이면 `n = ceil(dt/cap)`으로 나눠 재귀
호출한다.

---

## 1. 노심 중성자 물리

### 1.1 형상 상수

| 상수 | 값 | 의미 |
|---|---|---|
| `NX`, `NY` | 8, 8 | 반경 격자 |
| `NZ` | 12 | 축방향 슬라이스 |
| `N_CHANNEL` | 60 | 활성 채널 (8×8에서 네 모서리 제외) |
| 총 노드 | 720 | 60 × 12 |
| `CORE_HEIGHT` | 3.81 m | 연료 유효길이 (150 in, NRC 2.2.2.1 · Fig 3.1-1) |
| `NODE_H` | 0.3175 m | 슬라이스 두께 (`CORE_HEIGHT`/12) |
| `BUNDLES_PER_CHANNEL` | 12.73 | 한 채널이 대표하는 실기 다발 수 (764/60) |

`ACTIVE`는 (8,8) 불리언 배열로 네 모서리만 False. `mask`는 이걸 12층으로
복제한 (8,8,12) 배열이다. 모든 노심 계산은 `mask`가 True인 노드만 대상으로 한다.

**제어봉 인출 순서** `ROD_SEQUENCE`: 활성 채널을 (a) 체커보드 패리티
`(x+y)%2`, (b) 중심에서 먼 순서로 정렬한다. 즉 **바깥에서 중심으로** 뽑는다.
순서는 **A·B 두 벌**이고 어느 체커보드 색부터 뽑는가만 다르다 (§3.3c).
모듈 상수는 A 이며, 원자로는 자기 것을 `rod_sequence` 로 들고 있다.
반경 출력은 원래 중심이 높으므로 중심 제어봉을 마지막까지 남겨 눌러야
출력이 평탄해진다. 중심부터 뽑으면 중앙 채널이 200%를 넘는 편중이 생긴다.

> **중간 상태에서는 반경 분포가 뒤집혀 보인다.** 기동이 막 끝난 시점(제어봉
> 평균 90% 남짓)에는 중심 제어봉이 아직 들어가 있어 **최솟값이 노심 중앙**에
> 생기고 최소비가 0.59 까지 내려간다. 제논이 쌓이며 제어봉이 더 뽑히면
> (평형에서 평균 98.4%, 중심 4개만 65~88% 삽입) 최솟값이 **모서리**로 옮겨가
> 0.80 이 되고 중심이 가장 뜨거운 실기 모양으로 돌아온다. 지표를 인용할 때
> 어느 상태인지 밝혀야 하는 이유다(CHANGES §6-7).

### 1.2 노드별 반응도 `_reactivity()` → `self.rho` [pcm]

각 노드의 반응도를 아래 항의 합으로 만든다 (전부 pcm 단위):

```
rho = EXCESS_REACTIVITY                        (+23281)  잉여반응도
    - ROD_WORTH * rod_frac                     (-29500 × 점유율)  제어봉
    + DOPPLER_COEF * (fuel_temp - 20)          (-2.5/℃)  연료온도(도플러)
    + MODERATOR_COEF * (node_coolant - 20)     (-30/℃)  냉각재온도
    + VOID_COEF * void                         (-50/%)  보이드
    + XENON_WORTH * (xenon / XE_EQ_FULL)        (-2600 × 제논분율)
    + SM_WORTH * (samarium / SM_EQ_FULL - 1)    (-570 × 사마륨 편차, §1.7b)
    + BORON_WORTH * boron_core                  (-10/ppm)  붕산
    + FUEL_ZONE                                (반경 연료 존, 아래)
    + fuel_load                                (이번 장전의 채널 편차, §1.2b)
    + cycle_reactivity(exposure)               (연소도, §1.8)
```

- `EXCESS_REACTIVITY`(23281)와 `ROD_WORTH`(29500)는 **실제 설계값이 아니라**
  정격 조건(청정노심 제어봉 약 78% 인출, 평형제논 약 84%)이 맞도록 역산해
  굳힌 값이다. 이 둘은 별개 수치다 (잉여반응도 ≠ 제어봉 전량가치).
- 반응도 계수(도플러·감속재·보이드)는 pcm/단위의 실측 대표값이다.
- `FUEL_ZONE`: (8,8) 배열. 중심은 음수(낮춤), 외곽은 양수(높임)로 채널 출력을
  평탄화한다. 평형제논 상태(정격 30시간 운전 후)의 실제 출력 분포에서
  역산해 굳혔다. 발전소가 대부분의 시간을 보내는 조건이 그쪽이기 때문이다.
- `fuel_load`: (8,8) 배열. **시작할 때마다 다시 뽑는** 채널별 장전 편차다.
  `FUEL_ZONE` 이 설계 존이라면 이쪽은 이번 재장전의 사정이다 (§1.2b).

**`rod_frac` (노드별 제어봉 점유율 0~1)** — `_update_rod_frac()`:
제어봉은 하부에서 삽입되고, 인출하면 **상단부터** 비워진다.
```
uncovered = (rods/100) * NZ          채널별 인출된 슬라이스 수
j_from_top = NZ-1-arange(NZ)         위에서부터의 층 번호
rod_frac = clip(j_from_top + 1 - uncovered, 0, 1)
```
예: 16% 인출 → 최상단 슬라이스 0%, 그 아래 40%, 하부는 100% 점유.


### 1.2b 다발 장전 편차 `fuel_loading(seed)` → `self.fuel_load` [pcm]

`FUEL_ZONE` 은 반경 존 하나로 굳힌 배열이라 **노심이 매번 똑같다.** 자동
기동을 돌리면 인출 순서가 늘 같고, 정격에서 끝까지 안 뽑히고 남는 구역도
늘 같은 자리다. `RodAutoControl._move_rods` 가 그때그때 **출력이 낮은
채널부터** 고르기 때문이다 — 노심이 같으면 그 순서도 같다.

실기는 그렇지 않다. 재장전마다 신연료·1주기·2주기 다발을 섞어 배치하고
그 배치도가 주기마다 다르다. 어느 제어봉이 마지막까지 남는지도 주기마다
다르다. 모델에는 배치도가 없으므로 채널마다 장전 반응도를 조금씩 흩어
그 차이를 대신한다.

```
fuel_load[x,y] ~ U(-FUEL_LOAD_SCATTER, +FUEL_LOAD_SCATTER)   활성 채널만
rho += (FUEL_ZONE + fuel_load)[:, :, None]                   _reactivity()
```

`FUEL_LOAD_SCATTER` = **200 pcm**. **근거 없는 값이다**(NO BASIS). 두 가지로
묶어 정했다.

- **크기** — `FUEL_ZONE` 자신이 이미 갖고 있는 비대칭보다 한 자리 작다.
  이 배열을 8중 대칭(4회전 + 반사)으로 평균 내 빼 보면 편차가 **rms
  2042 pcm, 최대 6512 pcm** 이다. 역산으로 굳힌 배열이라 그만큼 울퉁불퉁
  하다. 새로 넣는 흔들림은 이미 있는 흔들림보다 작아야 한다.
- **결과** — 뽑은 배치를 그냥 쓰지 않는다. **첨두를 재서 조건을 벗어나면
  버리고 다시 뽑는다.** 값이 아니라 이 조건이 기능의 계약이다.

#### 장전 잣대 `loading_peak(load)`

냉간·전인출·되먹임 없는 정적 형상의 **노드 첨두**다. 그 조건에서 노드
반응도는 `EXCESS_REACTIVITY + FUEL_ZONE + load` 하나로 정해진다 —
도플러·감속재는 20℃ 기준이라 0, 보이드·제논·붕산도 0, 사마륨은 평형이라
0 이다. 그래서 원자로를 통째로 세우지 않고 확산-증배 반복만 300 번 돌려
바로 푼다(19 ms). 노심 하나를 뽑는 데 평균 세 번쯤 돌아 59 ms 이고,
씨앗 없는 `Reactor()` 는 한 번도 안 돈다.

확산은 `diffuse_field(f, mask)` **한 곳에만** 적어 두고 `Reactor._diffuse`
가 같은 것을 부른다. 두 벌을 두면 한쪽만 고쳐질 때 조용히 갈라진다.
갈라지지 않았음은 시험이 직접 잰다 — 같은 노심을 실제 원자로로 냉간
전인출에서 수렴시킨 값과 **2e-5 안에서 일치**한다.

#### 받아들이는 조건

```
FUEL_LOAD_PEAK_MIN (1.00) <= loading_peak(load) / loading_peak(0) <= FUEL_LOAD_PEAK_MAX (1.05)
```

분모가 **지금 물리로 잰 공칭 노심**이므로 상수가 낡지 않는다. 물리를
바꾸면 양쪽이 함께 움직인다.

- **상한 1.05** — 여유를 크게 둔 난간이다. 냉간 첨두비와 평형 총첨두의
  관계를 360 개 표본(S = 100·200·300 pcm)으로 재면
  `총첨두/공칭 = 1.1415 × (냉간비 − 1) + 1.0`, 상관 **0.983**, 잔차
  최대 0.29% 다. 1.05 는 평형 총첨두 **2.35** 에 해당해 8x8 설계 밴드
  상단 2.51 아래에 잘 들어온다. ±200 pcm 에서 이 상한에 닿는 추첨은
  실제로 없다 — 묶이는 쪽은 언제나 하한이다.
- **하한 1.00**, 곧 *공칭보다 평탄해지지 않는다*. 모델의 평형 총첨두
  2.2193 이 설계 밴드 바닥 2.21 의 **0.4% 위**에 앉아 있어서, '밴드 아래로
  내려가지 않는다' 와 '공칭보다 평탄해지지 않는다' 가 사실상 같은 조건이
  된다. 거르지 않으면 S = 200 pcm 의 평형 총첨두가 **2.1868~2.2428** 로
  퍼져 절반가량이 밴드 바닥 아래로 내려간다.

하한이 묶이므로 뽑히는 노심은 공칭과 같거나 **첨두가 조금 더 선 쪽으로만**
간다. 안전한 쪽이 아니라 한계에 가까운 쪽이므로, 열적 제한은 짐작하지 않고
실제 자동 기동으로 확인했다 (VERIFICATION §13.24).

#### 어디서 뽑는가

`Reactor(fuel_seed=None)` 이 **기본이고 편차가 없다** — 시험과 스냅샷은
늘 같은 노심이어야 하기 때문이다. 편차를 쓰는 것은 사람이 켜는 입구
둘뿐이다.

```
panel/bwr_panel.py    Panel.__init__      new_fuel_seed() — 제목에 8자리로 띄운다
run_core.py                               new_fuel_seed() — 첫 줄에 찍는다
```

같은 노심을 다시 보려면 그 씨앗을 넘긴다
(`python -m panel.bwr_panel 1A2B3C4D`, `python run_core.py 1A2B3C4D`).

#### 무엇이 달라지고 무엇이 안 달라지는가 — 실측

자동 기동을 냉간에서 정격(99%)까지 다섯 번 돌려 봤다 (공칭 + 씨앗 4 개).

| | 공칭 | 씨앗 1 | 씨앗 2 | 씨앗 3 | 씨앗 4 |
|---|---|---|---|---|---|
| 전인출 대수 | 40 | 40 | 40 | 40 | 40 |
| 덜 뽑힌 채널 | 20 | 20 | 20 | 20 | 20 |
| 총첨두 | 2.4447 | 2.4471 | 2.4478 | 2.4380 | 2.4564 |
| MAPRAT | 1.0703 | 1.0742 | 1.0761 | 1.0702 | 1.0769 |

**깊이는 달라졌다** — 같은 채널이 92.3% 였다가 95.0% 가 되는 식으로 최대
3 %p 움직였고, 총첨두도 ±0.4% 흔들렸다. 어느 채널이 가장 뜨거운가도 바뀐다.

**그런데 '덜 뽑힌 구역' 자체는 다섯 번 모두 같았다.** 깊게 남은 12 개는
언제나 `RWM_GROUPS[4]` 전체이고, 92% 언저리에 남은 8 개도 늘 같은 자리다.

이유는 연료가 아니라 **인출 순서**에 있다. 자동 인출은 실기 절차대로
**현재 RWM 그룹을 끝내고 다음으로** 간다(R-304B 7.5.2.1.2). 그래서 정격에
닿는 순간 덜 뽑혀 있는 것은 언제나 *마지막 그룹*이고, 그 그룹의 구성은
`RWM_GROUPS` 라는 고정 기하다. 장전을 아무리 흔들어도 그룹 **안에서의
순서와 깊이**만 바뀐다.

흩뿌림을 다섯 배(±1000 pcm)로 키워도 깊게 남는 12 개는 그대로였다.
달라진 것은 92~97% 언저리에 얕게 남는 쪽뿐으로, 한두 대가 들고 났다
(전인출 39·40·41 대). 그 대신 총첨두가 2.43 까지 내려가 밴드 바닥에
더 가까워졌다 — 크게 흔들수록 평탄해지는 쪽이라 얻을 것도 없다.

실기에서 '남는 구역' 이 주기마다 달라지는 것은 **BPWS 순서 자체를 바꾸기
때문**이다(A/B 두 벌). 그래서 순서도 두 벌로 만들었다 — §3.3c 의
'인출 순서는 두 벌이다'. 그쪽이 남는 구역을 통째로 옮긴다.

### 1.3 확산-증배 형상 `_kinetics(dt)`

**노드 증배계수**: `k_node = max(0.02, 1 + rho·1e-5)` (pcm → 절대값 환산)

**형상 반복** (`SHAPE_ITER` = 3회):
```
f = k_node * _diffuse(shape)
k_eff = Σf / Σshape          (활성 노드 합의 비)
shape = f / mean(f)           (평균 1로 정규화)
```

**`_diffuse(f)` — 6방향 확산**:
```
out = (1 - 4·COUPLE_R - 2·COUPLE_Z) · f       자기 잔류분 (자기계수 = 0.300)
    + COUPLE_R · (반경 4방향 이웃)             (0.13씩)
    + COUPLE_Z · (축 2방향 이웃)               (0.09씩)
축 경계: 없는 이웃 자리에 자기값 · REFLECT_Z (0.50) 를 넣는다
반경 경계: 그대로 누설
```

**축방향 반사체** `REFLECT_Z` = 0.50 — 실기는 노심 위아래에 물 플레넘이 있어
빠져나간 중성자의 상당수가 되돌아온다. 예전에는 이게 없어(맨 노심) 축 양끝이
0.51/0.30 까지 눌렸고, 형상을 평균 1 로 정규화하는 과정에서 **가운데가 그만큼
솟아** 노드 첨두가 2.194 까지 갔다. 총첨두로 환산하면 2.194 × `LOCAL_PEAKING`
= **2.479** 로 설계 총첨두 2.409(= 13.4/5.56)를 넘었고, 축 슬라이스 하나를 잘라
재는 APLHGR 이 그 왜곡에 그대로 노출돼 MAPRAT 만 1.0 을 넘었다.

0.50 을 넣으면 끝이 0.79/0.43 으로 올라오고 노드 첨두가 **1.961**(총첨두 2.216)
로 내려와 8x8 설계 범위(BWR/6 2.21 ~ BWR/5 2.51) 안에 든다. 고온채널 축첨두도
1.560 → **1.424** 로, 매뉴얼이 GEXL 시험 형상으로 드는 코사인 1.39(R-304B
Table 1.8-2)에 가까워진다. 상세는 CHANGES.md §5.25.

**반경에는 반사체를 두지 않는다.** 반경은 이미 `FUEL_ZONE` 이 실제 출력분포에서
역산돼 평탄화를 맡고 있고(§1.2), 반경 반사체는 둘레 채널이 60개 중 28개라
**+2021 pcm** 이나 되어 임계 조건을 통째로 다시 잡아야 한다.

> 반사체를 넣으면 누설이 줄어 k_eff 가 **+219 pcm** 오른다. 그래서
> `EXCESS_REACTIVITY` 를 23500 → **23281** 로 함께 낮춰야 정격 임계조건
> (제어봉 약 98.4% 인출)이 유지된다. 둘은 **한 쌍으로 움직인다.**
**자기계수 0.300**이 안정 하한이다 — 이보다 낮으면 노드가 자기 값을 남기지
않아 체커보드 진동이 생긴다. `COUPLE_R`(0.13)·`COUPLE_Z`(0.09)는 셀 폭
(반경 59 cm, 축 31.75 cm)의 결합 세기를 대표한다. 슬라이스를 30.5→31.75 cm 로
키우며 `COUPLE_Z` 는 유지했다 — `EXCESS_REACTIVITY`·`ROD_WORTH` 와 함께 정격에
맞춰 역산한 튜닝 상수라 단독으로 못 바꾼다. 그만큼 축방향으로 조금 더 섞여
`validate.py` 의 기동 직후 오프셋이 -9.9 → -7.9% 로 움직였으나, **운전 평형
상태(제논 평형·제어봉 98.4%)의 오프셋은 약 -14% 로 변경 전후가 같다.**

### 1.4 점동특성 (6군 지연중성자)

`_kinetics` 안에서 형상을 구한 뒤 노심 전체를 1점 점동특성으로 푼다.

**상수**:
- `BETA` = 0.0064 (지연중성자 총분율)
- `BETA_FRAC` = [0.033, 0.219, 0.196, 0.395, 0.115, 0.042] (군별 분율)
- `LAMBDA` = [0.0124, 0.0305, 0.111, 0.301, 1.14, 3.01] /s (군별 붕괴상수)
- `GEN_TIME` = 5e-5 s (중성자 세대시간)
- `SOURCE` = 1e-9 (중성자원 — 미임계에서도 검출 가능한 최소 출력 유지)

**노심 반응도**: `rho_core = (k_eff-1)/k_eff · 1e5` [pcm]

**즉발임계 판정**: `rho_core·1e-5 ≥ 0.95·BETA` 이면 `prompt_critical = True`.
반응도는 `min(rho_core·1e-5, 0.95·BETA)`로 잘라 즉발임계 폭주를 수치적으로 막는다.

**출력** (준정적 점동특성 해):
```
delayed = Σ(LAMBDA · precursor)                    선행핵종 붕괴 기여
core_power = (delayed + SOURCE/GEN_TIME) · GEN_TIME / (BETA - rho)
```
분모 `BETA - rho`가 반응도가 BETA에 가까워질수록 작아져 출력이 급증한다.

**노드 출력**: `power = core_power · shape` (형상으로 분배)

**선행핵종 갱신** (지수적분, 큰 dt에서도 안정):
```
eq = BETA_FRAC·BETA·core_power / (GEN_TIME·LAMBDA)   평형 선행핵종
d = exp(-LAMBDA·dt)
precursor = precursor·d + eq·(1-d)
```

### 1.5 출력 지표 (파생 속성)

- `total_power` (APRM 지시): `core_power·(1-DECAY_TOTAL) + decay_power`.
  붕괴열은 핵분열 출력에 **더하는** 게 아니라 그 일부(7.638%)가 지연된 것이므로,
  핵분열 즉발분에서 그만큼 빼고 붕괴열을 따로 더한다. 이중계산을 피한다.
- `channel_power()`: `power.mean(axis=2)` — 축방향 평균한 (8,8) 반경 분포.
- `axial_power()`: 반경 평균한 (12,) 축방향 분포.
- `axial_offset`: `(상반부-하반부)/(상반부+하반부)·100` [%]. 정격에서 -10~-20%
  (제어봉이 하부를, 보이드가 상부를 누르므로 중하부가 가장 뜨겁다).
- `lprm(level)`: LPRM 4레벨(A/B/C/D)이 노심높이 12.5/37.5/62.5/87.5%
  (`LPRM_NODES` = {A:1, B:4, C:7, D:10})에서 읽는 국부 출력. `LPRM_STRINGS`
  16개 위치의 평균.

### 1.6 붕괴열 `_decay_heat(dt)` (6군)

Wigner-Way 곡선 `0.066(t^-0.2 - (t+T0)^-0.2)`의 지수합 근사.

- `DECAY_FRAC` = [0.02915, 0.01823, 0.01116, 0.00802, 0.00329, 0.00653]
- `DECAY_TAU` = [2, 20, 200, 2000, 20000, 200000] s
- `DECAY_TOTAL` = 0.07638 (전 군 합 — 정격 대비 붕괴열 비중)

```
eq = DECAY_FRAC·DECAY_TAU·core_power     각 군의 평형 저장에너지
d = exp(-dt/DECAY_TAU)
decay_pool = decay_pool·d + eq·(1-d)
decay_power = Σ(decay_pool / DECAY_TAU)  현재 붕괴열
```

가장 긴 군(τ=200000 s ≈ 55시간)은 며칠 운전해야 포화한다. 그래서 사고
시험은 `mk_aged.py`로 5일 운전한 상태에서 시작해야 붕괴열이 실제와 맞는다.
**3군으로 줄이면 장기 꼬리가 24시간에 실제의 1/6로 식어버려 안 된다.**

> **검증할 때 기준을 맞춰야 한다.** ANS-5.1 참고값은 **무한조사**(충분히 오래
> 운전한 노심) 기준인데 `validate.py` 는 5일만 돌린다. 5일로는 위 마지막 군이
> 포화하지 못하므로 낮게 나오는 것이 당연하다. 그래서 `[19]` 는 같은 상수로
> 계산한 **포화값**(`Σ DECAY_FRAC·exp(-t/DECAY_TAU)`)을 함께 찍어 원인을 가른다 —
> 24시간에서 포화−ANS 가 **-14%**(군 적합), 5일−포화가 **-12%**(시험 기준)다.

### 1.7 제논·아이오딘 `_xenon(dt)` (3D 노드별)

활성 채널 60개 × 축방향 12층 = **720점**의 `node_iodine`·`node_xenon`을 저장한다.
각 점의 핵분열 출력 `power[x,y,z]`가 그 점의 생성과 소진을 결정한다.

```
dI/dt  = YIELD_I*p - LAMBDA_I*I
dXe/dt = YIELD_XE*p + LAMBDA_I*I - (LAMBDA_XE + XE_BURN*p)*Xe
rho[x,y,z] += XENON_WORTH * node_xenon[x,y,z] / XE_EQ_FULL
```

한 스텝의 출력을 고정하고 위 두 연립식의 해석해를 사용한다. 붕괴율이 같은
극한도 처리하며, 명시적 Euler 순서 오차와 음의 농도를 피한다. `I_EQ_FULL`,
`XE_EQ_FULL`과 기존 수율·반감기·소진계수·정격 제논 가치는 유지한다.
큰 스텝에서 출력 자체를 고정하는 근사는 남으므로 연립식의 정확해가 전체
중성자·열수력 결합의 큰 dt 정확도를 보장한다는 뜻은 아니다.

`iodine`·`xenon` 스칼라 API는 체적 평균을 반환한다. 스칼라 대입은 전체를
같은 농도로 초기화하는 호환 경로다. 구형 1점 저장상태는 그 평균을 유지하는
균일 분포로 옮기지만 과거 공간 이력은 복원하지 못한다. fixture는 다시 생성한다.
상단 계기·Recorder는 평균, LPRM 옆 Xe는 선택 채널의 A/B/C/D 높이 농도다.
이 Xe 표시는 중성자 검출기의 실측값이 아닌 계산값이다.

공간 제논에 따라 국부 반응도가 달라지므로 총출력이 같아도 AO·열한계가 변한다.
근거: NRC R-104B §1.12.7.1; 공간 효과 설명은
[DOE-HDBK-1019/2-93, NP-03 p.39](https://www.osti.gov/servlets/purl/10144945/).

### 1.7b 사마륨 `_samarium(dt)` (노심 1점) — NRC 1.12.7.2

```
promethium += (YIELD_PM·p - LAMBDA_PM·promethium)·dt
samarium   += (LAMBDA_PM·promethium - SM_BURN·samarium·p)·dt
```
- `LAMBDA_PM` = ln2/(53.08 h) = 3.627e-6 /s (프로메튬-149 붕괴)
- `YIELD_PM` = 0.0113 (질량수 149 사슬 수율)
- `SM_BURN` = `XE_BURN`/66 = 1.136e-6 (흡수단면적 비에서)
- `PM_EQ_FULL` = 3115.2, `SM_EQ_FULL` = 9944.0
- `SM_WORTH` = -570 pcm (평형 사마륨)

사슬은 `Nd-149 (1.7 h) → Pm-149 (53 h) → Sm-149 (안정)` 이다. Nd 는 Pm 보다
30배 빨리 붕괴하므로 따로 두지 않고 Pm 생성에 합쳤다.

**제논과 다른 점 둘.**
1. **안정핵종이라 붕괴항이 없다.** 중성자를 먹어야만(`-SM_BURN·samarium·p`)
   없어진다. 그래서 정지하면 소진이 멈추는데 Pm 은 계속 붕괴해, 사마륨이
   늘어난 뒤 **그대로 눌러앉는다.** 제논처럼 하루 만에 빠지지 않는다.
2. **평형 농도가 출력과 무관하다.** 평형에서
   `samarium = YIELD_PM/SM_BURN` 이라 `p` 가 소거된다. 출력을 바꿔도 평형값은
   같고 도달 속도만 달라진다.

**반응도는 평형을 0 으로 잡은 편차로 넣는다.**

```
rho += SM_WORTH * (samarium / SM_EQ_FULL - 1.0)
```

제논처럼 절댓값을 쓰면 안 된다. `EXCESS_REACTIVITY` 는 정격 임계조건에 맞춰
역산한 값이고 실기 노심에는 평형 사마륨이 들어 있으므로, 그 -570 pcm 은 이미
그 안에 있다. 절댓값으로 또 빼면 이중 계상이 되어 임계 조건이 어긋난다.
그래서 정격 운전 중 기여는 **정확히 0** 이고, 평형 위로 올라갈 때만 붙는다.

**초기값.** `samarium = SM_EQ_FULL`, `promethium = PM_EQ_FULL` — 둘 다 평형값이다.
평형에서 `samarium = LAMBDA_PM·promethium/(SM_BURN·p)` 이므로 사마륨이 평형이면
프로메튬도 평형이어야 앞뒤가 맞는다.

제논·아이오딘을 0 으로 두는 것과 어긋나 보이지만 그게 맞다 — 제논은 반감기
9시간이라 정지하면 정말 사라지고, 사마륨은 안정핵종이라 그대로 남는다.
`Pm` 만 0 으로 두는 안도 재 봤는데, Pm 이 차오르는 동안 사마륨이 평형 아래로
내려가 +57 pcm 이 붙어 운전 평형 열이 더 크게 틀어졌다(CHANGES.md §5.22 표).

`SM_WORTH` 는 임의 값이 아니라 제논에서 유도했다 — 농도비
`SM_EQ_FULL/XE_EQ_FULL` = 14.39 에 단면적비 1/66 을 곱하면 0.218 이고,
`XENON_WORTH`(-2600) × 0.218 = **-567 pcm**. 문헌의 평형 사마륨 가치
(-500~-700 pcm) 한가운데다.

정지 후 사마륨은 평형의 **1.313 배**(= (SM_EQ+PM_EQ)/SM_EQ)까지 올라가
**-179 pcm** 을 더한다. 96시간 시점이면 제논이 -21 pcm 으로 사실상 사라진
반면 사마륨은 -128 pcm 이다 — 며칠 뒤 재기동 여유를 결정하는 것은 사마륨이다.

### 1.8 연소도 `cycle_reactivity(efpd)` / `_burnup(dt)`

한 주기 `CYCLE_EFPD` = 500 유효전출력일(EFPD).

```
x = exposure / CYCLE_EFPD
cycle_reactivity = GAD_GAIN·x·exp(-3x) - DEPLETION_TOTAL·x²
                 = 3000·x·exp(-3x)     - 3000·x²
```
- 첫째 항(가돌리니아 소진): 초·중반에 반응도를 회복시킨다(가연성 독물이 탐).
- 둘째 항(연료 소모): 말기로 갈수록 반응도를 깎는다.
- 합쳐서 **주기 초·중반 평탄, 말기 급락** — 운전원이 주기 내내 제어봉을
  조금씩만 뽑아도 되는 실기 거동.

`_burnup`: `exposure += channel_power()·(dt/86400)`. 출력이 높은 채널이 빨리
타므로 시간이 지나면 스스로 평탄해진다(자기평탄화).

---


### 냉간에서 정격 평형까지 `RatedRun` — 제어반과 시험이 같이 쓰는 조리법

시험 fixture(`aged`)와 제어반의 '5일 운전' 버튼이 **같은 코드**를 부른다.
두 벌을 두면 한쪽만 고쳐질 때 화면과 시험이 갈라진다.

```
기동   RodAutoControl(target=1.0, heatup_limit=HEATUP_LIMIT)
       걸음 RATED_RUN_CLIMB_DT = 1 s
       출력 RATED_RUN_BYPASS_POWER(10%) 아래에서는 첨두 제한 우회
       RATED_RUN_CLIMB_TO(99.5%) 를 넘으면 유지로 — **걸어 둔다**
유지   걸음 RATED_RUN_HOLD_DT = 20 s, max_dt 도 20 s (성긴 걸음)
       RATED_RUN_HOLD_DAYS(5일) 만큼
```

**단계는 걸어 둔다.** 매번 출력으로 다시 판정하면, 유지 구간에서 출력이
잠깐 climb_to 밑으로 내려가는 순간 기동으로 되돌아가 유지 시간이 안 쌓인다
(실측: 20 분을 돌려도 '기동 30%'). 그러면 1 초 걸음과 20 초 걸음이 섞여
결과도 달라진다.

`advance(예산)` 는 **벽시계 예산**만큼만 돌고 돌려준다. tkinter 가 한
스레드라 제어반이 6 분을 잡고 있을 수 없기 때문이고, 시험은 같은 함수에
무한 예산을 준다. 끊는 방식이 결과를 안 바꾸는 것은 검사가 잰다.

> **제어반 tick 은 꺼져 있어도 다음 약속을 남긴다.** `tick_enabled` 가
> 거짓이면 물리를 안 돌리고 곧바로 돌아가지만, `root.after` 재예약은 한다.
> 예전에는 여기서 그냥 돌아가 연쇄가 끊겼고, 플래그를 다시 켜도 아무도
> tick 을 예약하지 않아 제어반이 영영 멈춘 채였다(§5.64). 시험은 이 플래그로
> 물리를 멈춰 두므로 결정성은 그대로다.

도달 상태(공칭 노심): 모의시간 5.40 일 · 출력 98.62% · 제어봉 평균 97.75% ·
제논 평형 · 연소도 약 5 EFPD. fixture 와 비트 단위로 같다.

## 2. 열수력 — 보이드·건도·냉각재온도

### 2.1 채널 열수력 `_channel_hydraulics(dt)`

**채널 유량** `channel_flow()` → (8,8):
```
wa = RECIRC_A_WEIGHT[:,None]              열별 A펌프 가중 (linspace 0.9→0.1)
pump = wa·recirc_a + (1-wa)·recirc_b       좌측열은 A, 우측열은 B가 주로 지배
nat  = natural_circ                        상수가 아니다 — 아래 참조
flow = nat + (1-nat)·pump
```
재순환 A/B 불균형이 좌우 채널 출력 경사를 만든다. `recirc_a`,`recirc_b`는
0.30(`RECIRC_MIN`)~1.00(`RECIRC_MAX`). 다만 이 둘은 **실제 속도**이고,
운전원이 잡는 것은 `recirc_demand_a/b`(지령)다 — §6.5a 참조.

**자연순환 몫은 보이드가 만든다** (§7.2.3.1.1, CHANGES §5.53):
```
natural_circ = NATURAL_CIRC · sqrt(void̄ / NATURAL_CIRC_VOID)
               clip 되어 [NATURAL_CIRC_MIN, NATURAL_CIRC] = [0.03, 0.30]

  void̄               노심 평균 보이드 [%] (직전 스텝 값)
  NATURAL_CIRC       0.30   정격 보이드에서의 몫
  NATURAL_CIRC_VOID  42     그 기준 보이드 [%] — 평형 노심 실측 42.9
  NATURAL_CIRC_MIN   0.03   보이드 0 에서 남는 단상 순환 (NO BASIS)
```
부력 유량 ∝ sqrt(구동수두)이고 수두 ∝ 보이드다. 상수 0.30 이던 시절에는
보이드가 없는 냉간 정지에서도 30% 가 흘렀다 — 구동력 없는 유량이었다.

보이드는 유량으로 정해지고 유량은 보이드로 정해지므로 서로 물려 있다.
**직전 스텝의 보이드**를 쓴다 — 실기의 되먹임도 같은 방향이라 자연순환
운전이 스스로 안정된다. 상한을 정격 몫에 두는 것은, 보이드가 정격을 넘는
상태는 이미 냉각이 나빠진 것이라 거기서 유량이 더 는다고 치면 모델이
낙관으로 기울기 때문이다.

**두 개의 30% 를 혼동하지 말 것.** `RECIRC_MIN` 0.30 은 **펌프 속도** 하한이고,
`NATURAL_CIRC` 0.30 은 **정격 보이드에서** 펌프가 0 일 때 남는 노심 유량이다.

| 펌프 속도 | 노심 유량 (정격 보이드) |
|---|---|
| 0% | 30% (자연순환) |
| 30% (`RECIRC_MIN`) | **51%** |
| 100% (`RECIRC_MAX`) | 100% |

validate `[8]`·`[4]` 가 "펌프 30.0% 유량 51.0%" 로 찍는 것이 이것이다.
출력이 낮으면 같은 펌프 속도에서 유량이 그보다 낮게 나온다.

**노드 엔탈피 누적** (채널을 따라 위로):
```
w = max(1, RATED_CORE_FLOW·flow / N_CHANNEL)    채널당 질량유량
q = node_total_power · RATED_MW·1e6/(N·NZ)       노드 발열 [W]
h = cumsum(q, axis=2)/w + core_inlet_h           엔탈피 [J/kg]
```

**건도** `x = clip((h - h_f)/h_fg, 0, 1)` (비등 시작 후 증가)

**보이드율** (드리프트 플럭스, Zuber-Findlay):
```
rho_g = pressure/(STEAM_Z·R_STEAM·(sat_temp+273.15))   증기밀도
G = RATED_CORE_FLOW·core_flow / CORE_FLOW_AREA          질량유속 [kg/m²s]
a = x / ( DRIFT_C0·(x + (1-x)·rho_g/rho_f) + rho_g·DRIFT_VGJ/G ) · 100
```
- `DRIFT_C0` = 1.10 (분포계수) — 거품이 유로 가운데(물살이 빠른 곳)로 몰려
  더 빨리 빠져나가는 것을 다룬다. BWR 8×8 봉다발 실측값(Ozaki & Hibiki).
- `DRIFT_VGJ` = 0.23 m/s (드리프트 속도, 기포류 대표값)
- `CORE_FLOW_AREA` = 8.09 m² — 다발당 105.9 cm²(채널 내경 134mm 정사각에서
  연료봉 62개×12.3mm를 뺀 값) × 764 다발. 정격 G ≈ 1620 kg/m²s로 실기
  (1500~2000)와 맞는다.
- `STEAM_Z` = 0.75 (증기 압축인자), `R_STEAM` = 461.5 J/kg·K

> **왜 슬립 모델에서 바꿨나**: 예전에는 "거품이 물보다 1.2배 빠르다"는 균질
> 슬립 모델(`a = 1/(1 + S·(rho_g/rho_f)·(1-x)/x)`, S=1.2)을 썼다. 노심 출구는
> 맞았지만(건도 14% → 보이드 73%, 실기 70~75%) **저건도 구간이 과대**했다 —
> 건도 1.7%에 보이드 22%, 3.7%에 39%로 치솟아 중간층이 부풀었고, 그래서 노심
> 평균이 48.4%로 실기 범위(35~45%)를 넘었다. 분포계수 C0가 이 구간을 눌러
> 준다. 바꾼 뒤 노심 평균 43.5%, 출구 67.5%, 출구건도 13.9%로 전부 실기
> 범위에 들어왔다. 유량 쪽은 원래부터 정상이었다(노심유량 13100 kg/s vs 실기
> 12600, 증기유량 1815 vs 1827).

**노출 보정 (중요)**: 물 위로 드러난 노드는 물이 없으니 사실상 전부 증기다.
엔탈피 계산만으로는 이걸 모르므로 `submerged`(잠김도, §2.3)로 보정한다:
```
a = a·submerged + VOID_MAX·(1-submerged)     VOID_MAX = 90
```
이게 없으면 노심이 절반 드러났는데 보이드가 76%에 머물러 임계가 유지되는,
물리적으로 틀린 일이 생긴다. 실제로는 물이 빠지면 보이드 반응도가 출력을 눌러야 한다.

**보이드 이완** (되먹임 안정용):
```
tau = VOID_TAU / max(0.3, flow)              VOID_TAU = 3.0 s
void += (a_target - void)·min(0.15, dt/tau)
```
보이드 되먹임은 이득이 1을 넘어(보이드↑→출력↓→보이드↓) 한 스텝에 다
반영하면 큰 dt에서 발산한다. `min(0.15, dt/tau)`로 한 스텝 반영량을 제한한다.

**냉각재온도** `node_coolant`: 비등 노드는 `sat_temp`, 미비등 노드는
과냉각액 온도 `min(t_sub, sat_temp)`. `exit_quality`는 채널 출구 건도의
활성채널 평균 (정격 약 14%).

### 2.2 연료·피복재 온도 `_fuel_temperature(dt)` — 멜트다운 경로

이 함수가 노심 노출·멜트다운의 핵심이다. 잠긴 노드와 드러난 노드를
**잠김도로 가중 혼합**한다.

**드라이아웃 판정**:
유효 압력·유량·수위에서는 각 노드의 실제 건도 x와 CISE 임계건도 x_cr를 비교한다.
CPR 탐색과 `_critical_quality_excess`를 공유하며 국부 질량유속과 비등길이를 사용한다.
```
excess = clip((x/x_cr - 1)/DRYOUT_TRANSITION, 0, 1)  # DRYOUT_TRANSITION = 0.10
degrade = 1 + (DRYOUT_PENALTY-1)*excess              # DRYOUT_PENALTY = 6
```
임계건도 초과에 따른 열전달 저하는 교육용 근사다. 실제 post-CHF 상관식 검증을
대신하지 않는다. 범위 밖에서는 기존 보이드 88%~VOID_MAX의 선형 저하를 사용한다.
국부 건도 판정은 매 스텝 적용하고, 5초 간격 MCPR 표시 캐시에 의존하지 않는다.

**잠김도** `submerged` (0=완전히 드러남, 1=완전히 잠김):
```
half = NODE_H·100/2                          슬라이스 반두께 [cm]
submerged = clip((water_level - NODE_LEVEL)/(2·half) + 0.5, 0, 1)
```
`NODE_LEVEL`은 각 슬라이스 중심 높이 [cm]. 수위가 슬라이스 중심보다 위면
잠긴 것. 슬라이스 두께에 걸쳐 부드럽게 변해 계단현상을 없앤다.

**연료 상단·하단의 기준**: `NODE_LEVEL = BOTTOM_OF_FUEL + (j+0.5)/NZ·CORE_HEIGHT·100`
이고, `TOP_OF_FUEL` = **-497.8 cm**(-196 in) 다. Figure 3.1-1 의 절대표고
(계기영점 517 in · 정상수위 554 in · TAF 358 in)를 정상수위 기준으로 환산한
값이다. 이렇게 두어야 `LEVEL_1`(-430.5)이 TAF 보다 67 cm 위에 와서, **저압
ECCS·ADS 가 노심이 드러나기 전에 걸린다**(NRC 3.1.3.1.7). 예전에는 -368 cm 를
써서 L1 도달 시 이미 17% 가 드러나 있었다. `BOTTOM_OF_FUEL` 은 여기서
`CORE_HEIGHT`(150 in) 를 뺀 **-878.8 cm** 로, Figure 3.1-1 의 BAF(208 in)와
정확히 일치한다.

**잠긴 부분 (wet)**: 냉각재가 열을 받아가 준평형 온도로 접근한다.
```
target = node_coolant + FUEL_DT_FULL·local·degrade    FUEL_DT_FULL=414
wet_move = (target - fuel_temp)·min(0.5, dt/FUEL_TAU)  FUEL_TAU=6 s
```
정격에서 연료 중심이 냉각재보다 약 414℃ 높다(`local`=노드출력분율).

**드러난 부분 (dry)**: 증기로만 식으므로 붕괴열이 그대로 쌓인다.
```
q_node = local·RATED_MW·1e6/(N·NZ)                     노드 발열 [W]
(+ 지르코늄 반응열, 아래)
cooled = STEAM_COOLING·(fuel_temp - sat_temp)          STEAM_COOLING=8.33 W/K
dry_move = (q_node - cooled)·dt/FUEL_NODE_CAP           FUEL_NODE_CAP=5.75e4 J/K
```

> **`STEAM_COOLING` = 8.33 W/K 의 근거**: 노드 표면적 약 0.83 m²(연료봉 67개분)
> × 증기 강제대류 h≈10 W/m²K. 증기는 물의 수십분의 일밖에 못 식힌다.
> (표면적이 노드 높이에 비례하므로, 유효길이를 144→150 in 으로 고치며
>  8.0 → 8.33 으로 함께 올렸다 — §5.15.)
> 이전 값 30 W/K(h≈38)는 실제 저유속 증기 열전달계수보다 2~7배 커서,
> 붕괴열 1%(노드당 45.7 kW)를 증기냉각만으로 1376℃에 눌러앉혔다 — 노심이
> 45% 드러났는데도 ECCS급 냉각원이 생겨 멜트다운이 안 일어나는 오류였다.
> 8.33 W/K면 붕괴열이 증기냉각을 크게 웃돌아, 노출 30% → 약 67분에 1200℃,
> 노출 47% → 약 93분에 연료 용융으로 이어진다(TMI-2·후쿠시마의 시간 척도).

**혼합**: `fuel_temp += submerged·wet_move + (1-submerged)·dry_move`
연료온도는 clip(20, 3200)로 제한.

**지르코늄-증기 반응** (물이 남아 증기가 있을 때만):
```
f = clip((fuel_temp - ZR_OX_ONSET)/(ZR_OX_FULL - ZR_OX_ONSET), 0, 1)   1200→1800℃
zr_heat = ZR_OX_MAX·f²                                   ZR_OX_MAX=46.9 kW/노드
q_node += zr_heat
```
피복재가 1200℃를 넘으면 증기와 반응하며 스스로 열을 낸다. 뜨거울수록 빨라져
붕괴열보다 큰 열원이 되고 온도가 폭주한다(TMI·후쿠시마의 수소 발생원).

**피복재 온도**:
```
clad_wet = node_coolant + CLAD_DT_NUCLEATE·local + CLAD_DT_DRYOUT·excess·max(local,0.05)
                          (25)                       (950)
clad_temp = submerged·clad_wet + (1-submerged)·fuel_temp·0.95
```
핵비등 중 피복재는 냉각재보다 25℃ 높을 뿐이다. 드러난 곳은 물막이 없어
연료 온도에 거의 붙는다(0.95배).

**손상·용융 판정**:
- `CLAD_DAMAGE_TEMP` = 1200℃ 초과 → `clad_damaged = True` (지르칼로이 급속산화)
- `FUEL_MELT_TEMP` = 2865℃ 초과 → `fuel_melted = True` (UO2 융점)

이 둘은 한 번 True가 되면 되돌아가지 않고, `rod_block`에서 제어봉 인출을 영구 차단한다.

### 2.3 압력용기 `_vessel_step(dt)` (내부 10분할)

`_vessel(dt)`는 이 함수를 `dt/10`으로 10번 부른다(압력 진동 안정).

**포화물성**: `sat_temp, h_fg, h_f = sat_properties(pressure)` — `SAT_TABLE`
(0.1~22 MPa) 선형보간.

**급수 지령** (자동 수위제어 시):
```
feed_setpoint = clip(steam_out + (level_setpoint - water_level)·8.0, 0, 2200)
```
증기유량 + 수위편차×8. 급수는 2차계통이 허용하는 만큼만: `demand =
min(feed_setpoint, bop.feed_limit)`, 1차 지연 `dt/3.0`로 실제 유량 접근.

**물 에너지 수지**:
```
wet = 1 - fuel_uncovered/100                    드러난 몫은 물을 안 데운다
q = total_power·RATED_MW·1e6·wet                 핵발열
  + feed_flow·CP_WATER·(feedwater_temp - water_temp)   급수 현열
  - AMBIENT_LOSS·(water_temp - dw_temp)          주위 열손실 (3900 W/K)
water_temp += q·dt/(water_mass·CP_WATER)         CP_WATER=4400
```
`AMBIENT_LOSS`가 없으면 정지한 원자로가 영원히 286℃를 유지한다. 정격온도에서 약 1 MW.

**비등**:
```
if water_temp > sat_temp:
    surplus = (water_temp - sat_temp)·water_mass·CP_WATER
    boil_rate = surplus/h_fg/dt                  증기 발생 [kg/s]
    boil_rate = min(boil_rate, 가용 물질량/dt)     없는 물은 못 끓인다
    water_temp = sat_temp
```

### 2.4 압력조절 (EHC) — 전원 의존성 포함

**제어전원 상실 시 (fail-safe)**: `auto_pressure and not control_power`이면
EHC 유압을 잃어 가감밸브·바이패스밸브가 스프링으로 닫힌다:
```
close = dt/BYPASS_STROKE
turbine_valve = max(0, turbine_valve - close)
bypass_valve = max(0, bypass_valve - close)
```
전원이 있을 때만 아래 정상 EHC 로직이 돈다.

**정상 EHC** (`auto_pressure and control_power`):
```
ff = boil_rate/RATED_STEAM_FLOW·VALVE_NOMINAL         전향 유량항
err = (pressure - PRESSURE_SETPOINT)/PRESSURE_SETPOINT   압력편차
press_int += err·0.05·dt        (안티와인드업: 밸브 포화 방향으론 적분 중단)
demand = max(0, ff + press_int)                         압력조절기 출력
turb = min(demand, load_limit)  (터빈 온라인일 때만, 아니면 0)
turbine_valve = clip(turb, 0, 1)
want = clip((demand - turbine_valve)/BYPASS_K, 0, 1)    남는 지령이 바이패스로
bypass_valve += clip(want - bypass_valve, -dt/BYPASS_STROKE, +dt/BYPASS_STROKE)
```
- `VALVE_NOMINAL` = 0.85 (정격 가감밸브 개도 — 100%에 붙이면 밸브 포화로 압력 못 잡음)
- `BYPASS_K` = 0.255 = `VALVE_NOMINAL·BYPASS_CAPACITY` (바이패스 전개 시 상당 개도)
- `BYPASS_STROKE` = 4 s (밸브 행정시간)
- EHC 로직: 가감밸브가 먼저 받고 남는 지령이 바이패스로 (NRC 3.2.2.4 Valve Control Unit —
  원문 "the bypass valve demand is established by subtracting the pressure/load LVG output
  from the pressure control unit output". 3.2.2.2 는 Load Control Unit 이라 다른 절이다)

### 2.5 SRV (안전방출밸브)

**밸브 구성** `SRV_GROUPS` = 13대를 4군으로:
| 설정압 | 대수 |
|---|---|
| 7.71 MPa | 4 |
| 7.79 MPa | 3 |
| 7.86 MPa | 3 |
| 7.93 MPa | 3 |

**개폐 로직** (밸브마다):
```
if 개방중:  pressure < 설정압 - SRV_BLOWDOWN 이면 닫힘   (재폐차압 0.12 MPa)
if 폐쇄중:  pressure > 설정압 이면 개방
단, 마지막 상태변경 후 SRV_MIN_DWELL(8초) 안에는 안 바뀜   (채터링 방지)
```
**safety 모드는 전원 무관**(고압 증기가 파일럿을 스프링에 대항해 직접 연다,
NRC 2.5.2.1). ADS 모드만 전원·공기가 필요(§4.4).

**방출량**: `srv_flow = srv_count·SRV_FLOW_EACH·pressure/RATED_PRESSURE`
(`SRV_FLOW_EACH` = 110 kg/s, 임계유동이라 압력에 비례).

### 2.6 증기 출구·압력·수위

**주증기 출구**:
```
opening = (turbine_valve + BYPASS_K·bypass_valve)·msiv      MSIV는 직렬로 곱함
break_steam = break_size·RATED_STEAM_FLOW·pressure/RATED_PRESSURE   파단 유출
driver_steam = HPCI_STEAM_RATE·(hpci_flow/HPCI_FLOW) + RCIC_STEAM_RATE·(rcic_flow/RCIC_FLOW)
steam_out = opening·VALVE_K·pressure + srv_flow + break_steam + driver_steam
```
- `VALVE_K` = 3.057e-4 = `RATED_STEAM_FLOW/RATED_PRESSURE/VALVE_NOMINAL`
- MSIV(0~1)가 상류 차단이므로 터빈·바이패스·파단 유량에 곱해진다.
- HPCI·RCIC 구동증기(`driver_steam`)도 압력용기에서 실제로 빠져나간다.

**물질량**: `water_mass += (feed_flow - boil_rate - letdown)·dt` (하한 MIN_WATER_MASS).
`letdown`은 RWCU 취출(수위가 연료 위일 때만).

**넘침**: 수위가 `OVERFLOW_LEVEL`(500 cm)을 넘으면 초과분이 주증기관·파단으로
억제수조로 빠진다(저압 ECCS 대량주입 시 압력용기가 가득 참).

**증기질량·압력**:
```
steam_mass += (boil_rate - steam_out)·dt
sv = VESSEL_VOLUME - water_mass/rho_water            증기 공간
pressure = STEAM_Z·steam_mass/sv·R_STEAM·Tk + gas_mass·R_GAS·Tk/sv
```
증기 분압 + 비응축가스(질소) 분압. `gas_mass`는 냉간에 갇힌 공기.

**수위** `water_level` [cm, 정상수위 기준 0]:
```
v = water_mass/rho_water + CORE_COOLANT_VOLUME·avg(void)/100    노심 기포 부피 포함
ref = WATER_VOLUME_NOMINAL + CORE_COOLANT_VOLUME·NOMINAL_VOID/100
water_level = (v - ref)/VESSEL_AREA·100
```
- `VESSEL_AREA` = 30 m², `WATER_VOLUME_NOMINAL` = 203.5 m³, `NOMINAL_VOID` = 44%
- 노심 기포도 수위를 밀어올린다. 스크램으로 기포가 꺼지면 수위가 먼저
  '축소(shrink)'했다가 급수로 회복 — 실기 대표 거동.

**TAF 아래는 다른 식을 쓴다** (CHANGES §5.58):
```
lv >= TOP_OF_FUEL  ->  위 식 그대로
lv <  TOP_OF_FUEL  ->  frac = (v - v_min)/(v_taf - v_min)      1=TAF, 0=바닥
                       level = TOP_OF_FUEL + (TOP-BOTTOM)·(frac-1)
```
`VESSEL_AREA` 는 **정상수위 근처의 계기 감도**로 잡은 값이다. 노심 구역까지
균일 단면으로 외삽하면 기하가 안 맞는다 — 정격 재고 218 m³ 를 30 m² 에 펴도
7.27 m 인데 BAF 는 정상수위 아래 8.79 m 다. 그래서 용기를 **다 비워도** 수위가
-723.9 cm 에 멈춰 `fuel_uncovered` 가 59% 에서 더 안 올라갔다(실측). 노심에는
연료·슈라우드가 자리를 차지해 같은 높이에 담기는 물이 적으므로, 그 구간은 남은
재고 비율로 TAF~BAF 를 잇는다. 바닥나면 BAF, 곧 노출 100% 다.

L1~L8 설정치는 **전부 TAF 위**라 스크램·ECCS 동작과 수축·팽창 거동은 위 식
그대로다. TAF 아래 보간은 **NO BASIS** — 기하 일관성을 맞추는 장치이지
열수력 모형이 아니다.

**노심 입구 엔탈피** (급수 + 회수수 혼합) — 포화액 기준 상대값:
```
subcool = max(0, sat_temp - water_temp)          벌크수 과냉각도 [K]
h_bulk  = max(0, h_f - CP_WATER·subcool)         회수수 엔탈피
sub_fw  = max(0, sat_temp - feedwater_temp)      급수 과냉각도
h_fw    = max(0, h_f - CP_WATER·sub_fw)          급수 엔탈피
core_inlet_h    = min(h_f, (wf·h_fw + (wc-wf)·h_bulk)/wc)
core_inlet_temp = min(sat_temp, sat_temp - (h_f - core_inlet_h)/CP_WATER)
```
정격에서 급수는 노심유량의 13.9%이고 71K 과냉각이므로, 혼합 과냉각도는
약 9.8K가 된다(실기 10~12K). 입구 엔탈피 1225 kJ/kg (실기 약 1230).

> **왜 `CP_WATER·T` 를 안 쓰나**: 예전에는 `h_bulk = min(h_f, CP_WATER·water_temp)`
> 였다. 이 근사는 저압에서 실제 포화액 엔탈피보다 20~28 kJ/kg 크게 나와
> (0.1~2 MPa 구간), 물이 포화온도보다 5℃ 낮은 과냉각 상태인데도 `min` 에 걸려
> **입구가 포화액으로 계산**됐다. 그러면 붕괴열이 조금만 들어와도 곧바로 건도가
> 생기고, 저압에서는 증기 비체적이 커서 건도 0.1%가 보이드 30~45%로 증폭된다.
> 그 결과 정지냉각 중에 있지도 않은 기포가 끓어오르며 진동했다. 포화액 기준
> 상대값으로 바꾸니 정지냉각 중 보이드가 0%로 안정됐다.

---

## 2.5 열적 제한 `_thermal_limits(dt)` — MCPR · LHGR · APLHGR

§2 가 구한 노드별 건도·출력에서 곧바로 나오는 감시량이다. 되먹임에는 쓰이지
않는다. 셋 다 **피복재를 뚫지 않는 것**이 목적이고, 막으려는 고장이 다르다.

| | 막는 고장 | 한계 조건 | 재는 것 |
|---|---|---|---|
| **LHGR** | 펠릿 팽창으로 피복재가 갈라짐 | 피복재 1% 소성변형 | 노드 안 최대 봉 |
| **APLHGR** | LOCA 후 냉각 부족 파손 | 피복재 2200°F | 다발 평면 평균 봉 |
| **MCPR** | 핵비등 상실로 과열 | 비등천이 개시 | 다발 전체 |

### 2.5.1 선출력 — LHGR · APLHGR

노드 하나(채널 × 축슬라이스)는 `BUNDLES_PER_CHANNEL` = 12.73 다발을 대표한다.
그 노드의 **다발 하나분** 열출력을 봉 수와 길이로 나눈 것이 APLHGR 이다.

```
q_node   = node_total_power() · RATED_MW / (N_CHANNEL · NZ)   [MW]  ← 붕괴열 포함
q_bundle = q_node / BUNDLES_PER_CHANNEL
APLHGR   = q_bundle / (RODS_PER_BUNDLE · NODE_FT)             [kW/ft]
LHGR     = APLHGR · LOCAL_PEAKING
```

- **`node_total_power()` 를 쓴다** (핵분열 출력 `power` 가 아니라). 실제로 연료를
  데우는 것은 붕괴열까지 합한 값이다. 붕괴열은 노심에 균일하게 더해지므로
  노드 첨두가 2.185 → 2.096 으로 조금 평탄해진다.
- **`LOCAL_PEAKING`(1.13) 이 구조적 간극을 잇는다.** LHGR 은 정의상 **봉 하나**의
  값인데 모델은 다발 내부를 나누지 않는다. 그래서 평면 평균(APLHGR)에 이 계수를
  곱해 최대 봉을 추정한다. 매뉴얼에 설계값이 없는 값이다.
- MAPLHGR 한계는 노심유량이 정격의 70% 이하면 95% 로 낮아진다(실기 규칙).

정격에서 노심평균 LHGR = 3293 MW / (764 다발 × 62 봉 × 12.5 ft) = **5.56 kW/ft**.

### 2.5.2 임계출력비 — CPR

비등천이(핵비등 → 천이비등)가 시작되는 다발 출력이 **임계출력**이고,
그 실제출력 대비 비가 CPR 이다. 노심 전체에서 가장 낮은 값이 MCPR 이다.

**임계건도 vs 비등길이** 형태를 쓴다. 비등길이 `L_B` 는 포화가 시작되는 높이
(건도 0 이 되는 지점)부터 그 위치까지의 거리다.

```
x_cr(L_B) = K · a·L_B / (L_B + b)          (CISE-4 형)
      a = (1 - P/P_c) / (G/1000)^(1/3)
      b = 0.199 · (P_c/P - 1)^0.4 · G · D_h^1.4
```

- 압력이 임계압에 가까울수록 `a` 가 작아져 임계건도가 **낮아진다**.
- 질량유속 `G` 가 크면 임계건도는 **낮아지지만**, 임계출력은 유량에 비례하므로
  결국 `G^(2/3)` 로 **올라간다**.
- `K`(0.638)는 원관 상관식을 다발에 맞추는 보정계수다. 정격에서 MCPR 1.45 가
  나오도록 역산한 튜닝 상수이며, `EXCESS_REACTIVITY`·`ROD_WORTH` 와 같은 성격이다.

**푸는 법은 접선법이다.** 다발 출력을 `k` 배 해 가며 열수지 건도곡선을 다시 그리고,
그것이 임계건도 곡선에 **처음 닿는** `k` 를 찾는다.

```
for 이분법 CPR_ITER(14)회:
    h(z) = h_in + Σ q·k / w          엔탈피 누적
    x(z) = (h - h_f) / h_fg          건도
    z_sat = x 가 0 을 지나는 높이 (노드 경계에서 선형보간)
    L_B(z) = max(0, z - z_sat)
    닿았나 = max( x(z) - x_cr(L_B(z)) ) > 0
    닿았으면 hi = k, 아니면 lo = k
CPR = k
```

채널 60개를 배열로 한꺼번에 푼다. 한 번에 약 4.4 ms이고, `THERMAL_UPDATE_DT`
(5초) 간격으로만 부르므로 스텝당 부담은 **0.83 ms(12%)** 다. 배열이 768개뿐이라
비용은 계산이 아니라 numpy 호출 오버헤드가 지배하고, 그래서 반복 횟수를 줄이는
것이 그대로 속도가 된다(24→14 회로 줄여도 값은 소수 셋째 자리까지 같다).

- 이분법 범위는 `[0.2, 12]` 다. 하한이 1 이면 **이미 비등천이에 든 채널**의
  CPR(<1)이 전부 1.000 으로 뭉개진다. 탐색폭 11.8/2¹⁴ = 0.0007 이라
  소수 둘째 자리까지 띄우는 데 충분하다.
- 12 배를 해도 안 닿는 채널(저출력·정지 중)은 제한이 의미 없으므로 `CPR_MAX`(99).
- `L_B < 5 cm` 구간은 상관식 범위 밖이라 접선 판정에서 뺀다.

### 2.5.3 한계 대비 비율

실기 발전소 전산기가 띄우는 형태 그대로다. **셋 다 1.00 이 한계**다.

```
MFLPD  = 최대 LHGR   / 13.4 kW/ft
MAPRAT = 최대 APLHGR / MAPLHGR 한계
MFLCPR = MCPR 운전한계(1.44) / 실제 MCPR      ← 분자·분모가 뒤집힌다
```

MCPR 만 클수록 안전하므로, 비율로 만들 때 뒤집어 다른 둘과 방향을 맞춘다.

**CPR 을 계산하면 안 되는 구간이 있다** (R-304B 1.8.5.4). 압력 785 psig 미만
또는 노심유량 10% 미만에서는 GEXL 상관식이 유효하지 않아, 실기도 MCPR 을 쓰지
않고 **열출력 25%** 라는 다른 안전한계를 지킨다. 모델도 같게 해서, 유효범위 밖
(노심 노출 포함)에서는 MCPR 을 계산하지 않고 `MFLCPR` 칸에 열출력/25% 를 띄운다.

### 2.5.4 거동 확인

| 조작 | MCPR | 이유 |
|---|---|---|
| 유량제어선을 따라 출력 하강 (100→72% 유량) | 1.46 → **1.45** 거의 불변 | 출력과 유량이 **함께** 내려간다. BWR 의 특징 |
| 유량만 급감 (출력이 못 따라옴) | 1.46 → **1.35** | 분모는 그대로인데 냉각이 줄었다 |
| 출력만 1.6배 (합성) | 1.46 → **0.93** | 비등천이 안쪽 |

첫 줄이 상관식이 제대로 들어갔다는 가장 좋은 증거다. 이 특성은 맞추려고 튜닝한
것이 아니라 압력·유량·건도 의존성에서 저절로 나온다.

---

## 3. 보호계통 `_protection(dt)`

**스크램 시 제어봉 삽입**:
```
if scrammed:
    (스크램 걸린 그 스텝은 한 박자 유예 — 표시·배속조정이 삽입보다 먼저)
    rate = 100/ROD_INSERT_TIME              ROD_INSERT_TIME=4 s
    if (not scram_force_ok) or sdv_full:    미는 힘이 없거나 받을 자리가 없으면
        rate /= CRD_SLOW_FACTOR             4배 느리다  (§3.4)
    rods -= rate·dt
```
제어봉은 떨어지는 게 아니라 **밀려 올라간다.** 무엇이 미는지는 §3.4 (CRD).

**스크램 판정** (우선순위 순, `rps_enabled`일 때만):
1. 터빈 정지밸브 폐쇄 (`bop.trip_event` and 출력>`TSV_SCRAM_POWER`(0.30) and msiv>0.9)
2. 드라이웰 고압력 (`dw_pressure > DW_HIGH_PRESSURE` = 115.8 kPa)
3. 고압력 (`pressure > SCRAM_PRESSURE` = 7.38 MPa)
4. 저수위 L3 (`water_level < SCRAM_LEVEL` = -62 cm)
5. APRM 고출력 (`total_power > aprm_scram_setpoint`, 아래)
6. 터빈밸브 급폐 (수동 조작 + 급속폐쇄 + 출력>30%)
7. 단주기 (`0 < core_period < SCRAM_PERIOD` = 10 s)
8. **IRM 고고** (`irm_frac >= IRM_SCRAM` = 120/125) — 위 일곱 개가 아무것도
   안 걸리는 기동 구간(정격의 1e-5 ~ 수 %)을 지킨다. 모드스위치가 RUN 이고
   APRM 이 눈금 안이면 자동우회 (§3.2)

**APRM 유량바이어스 설정치**:
```
aprm_scram_setpoint = min(APRM_SCRAM_CLAMP, APRM_FLOW_SLOPE·flow% + APRM_SCRAM_BIAS)/100
                    = min(113.5, 0.66·flow% + 51)/100     스크램
aprm_block_setpoint = min(108.0, 0.66·flow% + 42)/100     제어봉 인출차단
```
- `APRM_FLOW_SLOPE` = 0.66 (실기 기동시험: 유량 10% 변화 → 출력 6.6% 변화)
- 유량이 낮으면 트립점도 내려간다(저유량은 열적 여유가 적음).
- 두 설정치는 **기울기 바이어스도 상한도 다르다**. NRC Table 5.4-1 이
  `APRM High = .66(W)+42%, 108% Max → Rod Block` 과
  `APRM High-High = .66(W)+51%, 113.5% Max → Scram` 을 별개 행으로 적는다.
- 그래서 차단선이 스크램선보다 항상 낮다 — 정격유량 108 vs 113.5%,
  그 아래 전 구간에서 9%p 차. 먼저 못 뽑게 막고, 그래도 오르면 세운다.

| 노심유량 | 인출차단 | 스크램 |
|---|---|---|
| 100% | 108.0 | 113.5 |
| 90% | 101.4 | 110.4 |
| 70% | 88.2 | 97.2 |
| 50% | 75.0 | 84.0 |

같은 표의 고정 118% 스크램은 유량바이어스 설정치가 113.5% 에서 잘려 도달할
수 없으므로 넣지 않았다.

**RBM (제어봉 차단 감시기)** `rbm_setpoint = 0.66·flow% + 41`:
절대 출력이 아니라 **선택한 제어봉 주변의 국부 출력 상승분**을 본다.
제어봉을 고르는 순간 `rbm_null()`로 영점을 맞추고(그 봉 주변 최대 채널출력을
APRM 지시와 같게), 그 뒤 뽑는 동안 얼마나 더 오르는지를 잰다. 국부 고출력은
스크램이 아니라 **인출차단**만 건다.

**`rod_block(channels)`** — 인출 차단 판정 (막힌 사유 문자열 또는 None):
1. `fuel_melted` → "연료 용융 — 인출 불가"
2. `clad_damaged` → "피복재 손상 — 인출 불가"
3. `slc_valves_fired` → "붕산 주입됨 — 인출 불가"
4. `scrammed` → "스크램"
5. `total_power > aprm_block_setpoint` → "APRM 인출차단"
6. RBM 지시 > `rbm_setpoint` → "RBM 인출차단"
7. `sdv_level > 0.05` → "배수용기 미배수" (§3.4)
8. 기동영역 계측기 `nms_rod_block()` — SRM·IRM 트립 (§3.2). APRM·RBM 은 출력영역
   계기라 임계 접근 구간에서는 아무것도 못 막는다. 그 구간을 막는 것은 여기다.

---

## 3.2 기동영역 중성자계측 `_nms(dt)` — SRM · IRM (NRC 5.1 · 5.2)

APRM 은 출력영역 계기라 기동 구간을 못 읽는다. 그 아래를 계기 둘이 나눠 덮고,
서로 **겹쳐서** 넘겨받는다.

```
SRM   계수율 [cps]   1e-1 ~ 1e6 (7 decade, 로그)      정지 ~ 임계 접근
IRM   10 레인지 x 반 decade, 레인지마다 선형 0~125     SRM 상단 ~ 출력영역 하단
APRM  %                                              출력영역
```

**읽는 양은 `core_power`(핵분열 출력)다.** `total_power` 는 붕괴열을 포함해서,
스크램 한 시간 뒤에도 0.96% 를 가리킨다 — 중성자 검출기가 볼 수 없는 값이다.
같은 시점 `core_power` 는 3.0e-09 이고 계수율 29 cps 로 냉간 수준이다.

```
검출기 자리의 속 = core_power · channel_power[x,y] / mean(channel_power)
SRM cps = clip(속 · SRM_CPS_PER_POWER · 10^(-SRM_RETRACT_DECADES·srm_pos),
               SRM_CPS_MIN, SRM_CPS_MAX)
IRM 지시 = min(1, 속 / irm_full_scale())
irm_full_scale(n) = IRM_FULL_R10 · 10^((n-10)/2)      반 decade 씩
```

**눈금의 근거**는 매뉴얼 5.2.3.3 의 두 문장이다. 절대 눈금은 "레인지 10 의
100/125 가 정격의 40%" 가 정하고, SRM 은 "계수율 1e4~1e5 이면 IRM 이 레인지 1
에서 온스케일" 이 정한다. `SRM_CPS_PER_POWER = 1e10` 이 유일한 자유상수이며
그 겹침에 맞춰 잡았다 (VERIFICATION §13.16).

| 레인지 | 만눈금 [정격분율] | | 레인지 | 만눈금 |
|---|---|---|---|---|
| 1 | 1.58e-05 | | 6 | 5.00e-03 |
| 5 | 1.58e-03 | | 10 | 5.00e-01 |

**인터록** (Table 5.1-1 · 5.2-1). 자동우회가 곧 계기 사이의 인수인계다.

| 트립 | 설정치 | 동작 | 자동우회 |
|---|---|---|---|
| SRM 다운스케일 | 3 cps | 인출차단 | IRM 레인지 > 2 |
| SRM 인출허가 | 100 cps | 인출차단(뺀 채 낮으면) | IRM 레인지 > 2 |
| SRM 업스케일 | 1e5 cps | 인출차단 | IRM 레인지 > 7 |
| IRM 다운스케일 | 5/125 | 인출차단 | 레인지 1 |
| IRM 업스케일 | 108/125 | 인출차단 | 모드스위치 RUN |
| IRM 고고 | 120/125 | **스크램** | RUN **그리고** APRM 온스케일 |

**자동 조작.** 레인지 스위치와 검출기 위치는 실기에서 운전원이 움직인다
(75/125 에서 한 칸 올리고, 계수율을 1e2~1e5 로 유지하며 검출기를 뺀다).
`irm_auto_range` · `srm_auto_retract` 가 기본 켜짐으로 그 절차를 대신한다.
자동 레인지의 내림 조건은 `지시·√10 < 75/125` 인데, 매뉴얼의 유지대역
25/125~75/125 이 정확히 한 칸(3.16배)이라 25/125 에서 그냥 내리면 내리자마자
79/125 가 되어 도로 올라가기 때문이다.

**모드스위치**는 'RUN 인가' 하나만 둔다(실기는 네 자리). 레인지 10 만눈금이
정격의 50% 이므로, RUN 으로 안 넘기면 정격의 48% 에서 IRM 이 스크램을 건다 —
실기도 같다. `RodAutoControl` 이 그 운전원 조작을 대신한다.

**옛 스냅샷**은 계측기 자리가 비어 있다. 기본값이 냉간 기동 배치라 출력 중인
스냅샷에 씌우면 복원 5 초 만에 스크램하므로, `__setstate__` 이 `_align_nms()`
로 지금 출력에 맞는 자리(레인지·검출기·모드)로 옮긴다.

## 3.0b 제어봉 고착과 정지여유 (NRC 2.3.1 · 1.12.2)

**고착** — `rods_stuck` 은 (NX, NY) 불리언이다. 스크램 삽입이 이 마스크를 뺀다.

```
moved = max(0, rods - rate·dt)
rods  = where(rods_stuck, rods, moved)
```

137 개 HCU 가 저마다 독립이므로(2.3.1) 하나가 실패해도 나머지는 정상 속도로
든다. 수동 인출·삽입도 고착된 봉은 비껴간다.

**정지여유** `shutdown_margin()` — 매뉴얼 정의를 그대로 따른다.

> "assumes that the **strongest control rod is stuck in the fully withdrawn
>  condition**" (1.12.2)

사본에서 활성 채널을 하나씩 100 % 로 두고 나머지를 전삽입해 k_eff 를 수렴시킨
뒤, 가장 높은 k_eff 를 고른다. `SDM = 1 - k_eff(최대)`.

| | k_eff | 여유 |
|---|---|---|
| 전삽입 | 0.8646 | 13.54 % |
| 가장 센 봉(ch34) 고착 | 0.8917 | **10.83 %** ← 정지여유 |

전삽입 값을 정지여유라 부르면 2.71 %p 낙관적이다. 60 칸 × 0.05 초 ≈ 3 초라
매 스텝이 아니라 **호출할 때만** 계산한다.

**노치는 두지 않았다.** 제어봉 위치는 0~100 % 연속값이다 — 실기의 25 개 래치
위치(6 인치 간격, 00~48)로 끊어도 반응도 적분이 같고, RWM 이 뱅크 위치를
강제하지 않는 지금 모델에서는 강제할 대상도 없다 (§5.47).

## 3.1b RPS 전원 `_rps_power(dt)` — 120 VAC (NRC 9.3.1.3 · 7.3)

스크램 솔레노이드는 평소 **여자된 채**로 공기를 붙들고 있고, 스크램이란 그것을
**소자**시켜 공기를 빼는 것이다. 그래서 전원을 잃으면 스크램이다 (7.3, fail safe).

```
급전 = (rps_mg[k] or rps_alt[k]) and bus_ok(RPS_BUS_AC[k])
급전이면  spin[k] = 1
아니면    spin[k] -= dt / RPS_MG_COASTDOWN(5 s)      하한 0
살아있음 = spin[k] > 0
```

**one-out-of-two-twice** (7.3.2.2):

| 죽은 모선 | 결과 |
|---|---|
| 1 | half scram — 그쪽 솔레노이드만 소자. 제어봉은 안 움직인다 |
| 2 | **스크램** |

대체전원은 MG 정비용이고, 9.3.1.3 의 인터록이 **두 차단기 동시 폐로**를 막는다
(`if all(rps_alt): rps_alt[1] = False`).

이 판정은 `rps_enabled` **바깥**이다 — 계장공기 상실과 같은 이유다 (§3.3 참조).

`RPS_MG_COASTDOWN` 은 NO BASIS 지만 결론을 흔들지 않는다. 갈리는 지점은 디젤
기동 시간(10 s)이고, 매뉴얼이 플라이휠을 "momentary" 변동용이라 하므로 그에
해당하는 값(1~5 s) 안에서는 LOOP 가 언제나 RPS 로 선다.

## 3.2b 24 VDC 계측전원 `_nms_power(dt)` (NRC 9.4)

§3.2 의 SRM·IRM 에 전기를 대는 계통이다. 배전모선 2 개, 모선마다 충전기 2 +
축전지 2, **비상 480 VAC** 에서 강압해 받는다(9.4.1).

```
급전 = 충전기 대수 > 0  and  bus_live[NMS_DC_BUS_AC[k]]
급전이면  charge += dt / (DC_RECHARGE_H·3600)      상한 1
아니면    charge -= dt / (NMS_DC_ENDURANCE_H·3600)  하한 0
살아있음 = 급전 or charge > 0
```

채널 i 는 `i % 2` 번 모선에 물린다 — 한쪽을 잃어도 절반이 남는다 (NO BASIS).

**이 계통은 스크램을 걸지 않는다.** 9.4.1 이 못 박는다: *"the instruments
supplied by the system do not trip the reactor or place the plant in a safe
shutdown condition."* 그래서 전원을 잃은 채널은

- `nms_rod_block()` 의 트립 비교에서 **빠지고**, 대신 '동작불능' 인출차단이 걸린다
- IRM 고고 **스크램** 비교에서도 빠진다

지시를 0 으로 읽어 다운스케일로 처리하면 이유가 틀린 차단이 나온다 — 계기가
낮은 것이 아니라 없는 것이다. 모드스위치가 RUN 이면 어차피 우회 구간이라
출력 운전 중에는 아무 일도 없다.

| 상수 | 값 | 근거 |
|---|---|---|
| `NMS_DC_BUSES` | 2 | 9.4.1 |
| `NMS_DC_CHARGERS` | 2 | 9.4.1 |
| `NMS_DC_BUS_AC` | (0, 1) | 비상모선이라는 것만 9.4.1, 어느 모선인지는 **NO BASIS** |
| `NMS_DC_ENDURANCE_H` | 8.0 | **NO BASIS** — 125 VDC 와 같은 값 |

## 3.3 계장·서비스 공기 `bop._air(dt)` (NRC 11.6)

수신기 하나에 압축기가 밀어 넣고 계통이 빼 쓴다.

```
공급 = min(보유대수, AIR_RUN_NORMAL + 대기기동) · AIR_EACH_SCFM(1000) · power_avail
소요 = AIR_DEMAND_NORMAL(1200) + air_leak       (서비스 격리 시 -400)
dP/dt = (공급 - 소요) / AIR_RECEIVER(1150)      clip[0, AIR_RCV_DESIGN]
```

압축기 부하 제어는 120/130 psig 히스테리시스다(11.6.3.1). **압축기는 정상
교류전원 부하**라(11.6.4.2) 전원을 잃으면 함께 멎고 디젤로 안 살아난다.

압력이 내려갈 때의 순서는 11.6.3.2 가 그대로 준다. 설정치는 그 순서가 나오도록
잡은 값이다(NO BASIS).

| 압력 | 일어나는 일 |
|---|---|
| 115 psig | 대기 압축기 자동 기동 |
| 100 psig | 서비스 공기 헤더 격리 → 소요 -400 scfm |
| 80 psig | `air_ok` False → **스크램** (스크램 파일럿 헤더 상실) |
| 60 psig | `air_msiv_ok` False → MSIV 를 못 잡는다 (스프링이 닫는다) |

MSIV 는 `want_open = msiv_demand and not msiv_isolated and bop.air_msiv_ok` 다.
격리 래치가 아니라 **동력 상실**이므로 공기가 돌아오면 다시 열린다.

**운전원 스위치는 이 판정들을 못 막는다.** `rps_enabled`·`msiv_auto`·
`auto_pressure` 는 **논리**를 끄는 스위치이지, 동력을 잃은 밸브가 안 닫히게
만들지는 못한다. 그래서 아래 셋은 스위치 밖에 있다.

    계장공기 상실 -> 스크램        `_protection` 의 rps_enabled 판정 **앞**
    제어전원 상실 -> MSIV 폐쇄     `_isolation` 의 msiv_auto 블록 **밖**
    제어전원 상실 -> EHC 밸브 닫힘  `auto_pressure` 와 무관

셋 다 한때 스위치 안에 갇혀 있었다 (§5.40).

**SRV·ADS 는 공기와 무관하다.** ADS 밸브마다 축압기와 체크밸브가 있어 공급을
잃어도 열고 유지한다(10.2.2.2). 공기 상실이 감압 수단까지 앗아 가면 fail-safe
가 아니라 공통원인 고장이 된다.


### 소내 전력 장부 — 누가 얼마를 먹는가

발전소가 쓰는 전기를 **한 장부**로 센다. 예전에는 둘이었고 서로 겹치지
않았다 — `bop.aux_mw` 는 2차계통 전동기만, `bus_load_kw` 는 비상모선
전동기만 셌고 송전출력은 앞의 것만 뺐다. 그래서 ECCS 를 전부 돌려도
소내부하가 움직이지 않았다.

```
aux_mw = 재순환 12 MW x (속도)^3 x 전원계수
       + 순환수 1.51 x 대수  + 냉각탑팬 0.25 x 대수
       + 복수 3.0 x 대수     + 부스터 1.60 x 대수
       + TBSW 0.352 x 대수   + TBCLCW 1.007 x 대수
       + 계장공기 0.160 x 대수 + 기계식 진공펌프 0.200
       + reactor_aux_mw       (아래)
       + AUX_HOUSE 잡부하
reactor_aux_mw = 급전되는 비상모선의 bus_load_kw 합 + RWCU 펌프
```

**두 값은 목적이 다르다.** `bus_load_kw(i)` 는 *그 모선에 매달린* 부하다 —
디젤이 떠맡을 양을 미리 알아야 하므로 급전 전에도 값이 있다. 소비로 세는
쪽(`reactor_aux_mw`)은 **급전되는 모선만** 본다. 이 구분을 안 하면 전소내
정전에서도 1.756 MW 를 먹는다(실측).

전동기 정격은 **교범이 주는 용량에서 역산**한다. 손으로 적은 kW 는 용량을
고칠 때 따라오지 않는다.

| 기기 | 교범 | 값 |
|---|---|---|
| CRD 펌프 | 2.3.2.1 200 gpm @ 1600 psig | 198.9 kW |
| SLC 펌프 | 39 gpm x 정격압 | 23.0 kW |
| RWCU 펌프 | 2.8.2.2 정격급수의 1% | 57.0 kW (두 대) |
| 계장공기 압축기 | 11.6.2.1 1000 scfm @ 125 psig, 3단 | 0.160 MW |
| 기계식 진공펌프 | 8.1.2.6 — 정격 없음 | 0.200 MW (**NO BASIS**) |
| 노심살수 / RHR / 서비스수 / 드라이웰 냉각기 | 기존 | 900 / 1400 / 454 / 300 kW |

RHR 펌프는 LPCI 주입만이 아니라 **정지냉각·수조냉각·살수로 돌 때도 같은
전동기**다. 예전에는 `lpci_on` 일 때만 셌다.

**급수펌프는 전기를 안 먹는다.** 주증기 구동 터빈이라(2.6.2.7) 축동력
`pump_mw` 를 주터빈 축일에서 뺀다. 대수를 줄이면 소내부하가 떨어져 보이는데
그것은 급수가 끊겨 재순환이 런백한 것이다(1.000 -> 0.900, 12 MW 의 세제곱).

실측: 정격 34.470 MW(총출력의 3.05%) · LOOP 2.056 MW(디젤이 떠맡은 몫) ·
전소내정전 0.000 MW.

**아직 없는 계통** — 교범 11.3 의 RBCLCW(원자로건물 폐회로 냉각수)가 모델에
통째로 없다. TBCLCW(터빈건물, 11.5)만 있다.






### 터빈건물 냉각수가 무엇을 붙들고 있나 (R-304B Table 11.5-1)

```
냉각탑 수조 -> [TBSW 11.4] -> 열교환기 -> [TBCLCW 11.5] -> 터빈건물 기기
```

이 계통이 식히는 것은 압축기만이 아니다. 개정판 Table 11.5-1 이 부하를
통째로 주고, 그중 **발전기 고정자 냉각**이 모델에 연결돼 있다.

```
TBCLCW 상실
  -> 고정자 냉각 열교환기 냉각 상실        (9.1.3.1.3)
  -> 70 초 뒤 터빈 트립                  (Table 3.2-1, 부하 25% 초과일 때)
  -> 터빈정지 스크램
동시에
  -> 60 초 뒤 압축기 정지                (11.6.4.2)
  -> 계장공기 감압 -> 110 / 105 / 95 psig 단계 -> 스크램 파일럿 헤더 상실
```

앞의 길이 훨씬 빠르다. 예전에는 뒤의 길만 있어서 8.5 분이 걸렸다(§5.69).

**실기는 트립 전에 한 단계가 더 있다** — EHC 런백 회로가 발전기 부하를 25%
미만으로 내린다(3.2). 압력추종 터빈인 이 모델에는 부하 제어 경로가 없어
경보로만 알린다.

### 계장·서비스 공기 단계 (R-304B 11.6.4.1)

```
125 psig  정상 — 1 대 연속운전, 3단 토출 PCV 가 119/125 로 언로드·로드
110 psig  1 차 예비 압축기 기동
105 psig  2 차 예비 압축기 기동
 95 psig  서비스 공기 헤더 격리 (AOV-010) — 계장용을 지킨다
 80 psig  스크램 파일럿 헤더 상실 -> 스크램        (NO BASIS)
```

한 번 뜬 압축기는 **압력이 돌아와도 안 선다** ("remains running until manually
shut off or tripped"). 구조는 옛 판 11.6.3.2 가, 수치는 개정판이 준다.

> **드라이웰 안은 질소다.** SRV 와 **내측** MSIV 는 격납 불활성화 계통의
> 질소로 움직인다 — 공기를 쓰면 드라이웰 누설이 산소 농도를 올려 불활성화를
> 스스로 깨기 때문이다(11.6.4.1). 그래서 계장공기를 잃어도 SRV 는 산다.
> 모델이 공기에 건 MSIV 는 **외측** 밸브에 해당하고, 둘 중 하나만 닫혀도
> 격리되므로 결과는 같다.

### 외부전원 두 갈래와 절체 (NRC 9.2.3.1 · 9.2.3.3)

```
정상원 NSST 138 kV (Main Bus)     <- 주발전기 분기 또는 계통 역송
예비원 RSST  69 kV (Startup Bus)
```

**고속절체는 한 방향뿐이다.** 모선 저전압을 보고 약 5 사이클 만에 정상원 ->
예비원으로 넘어간다. 교범이 "not in reverse" 라고 못 박으므로 반대 방향
자동 절체는 **없다**.

```
_offsite()          정상 -> 예비 자동 (고장 시, 5 사이클 전압 공백)
restore_offsite()   운전원 조작 — 두 가지를 겸한다
                      급전 없음    : 살아 있는 원에 붙인다 (정상원 우선)
                      예비원 급전중 : 정상원으로 되돌린다 (전압 공백 없음)
```

되돌리는 쪽에 공백이 없는 이유는 계획된 조작이기 때문이다 — 실기의 무정전
병렬 절체에 해당한다.

> **되돌리지 않으면 대가가 있다.** 예비원에 머문 채 그것을 잃으면, 정상원이
> 멀쩡해도 외부전원 상실이 된다. 역방향 자동 절체가 없어서다(§5.68).

### 수소 — Zr-증기 반응과 격납용기 (R-104B 4.1.3.3 · 4.1.3.7 — 개정판은 4.1.2.4.3 · 4.1.2.4.4)

피복재가 약 1200℃ 를 넘으면 증기와 반응한다.

```
Zr + 2 H2O -> ZrO2 + 2 H2 + 열
  ZR_H2_PER_ZR  = 2·M_H2 /M_ZR = 0.0442 kg H2  / kg Zr
  ZR_H2O_PER_ZR = 2·M_H2O/M_ZR = 0.395  kg H2O / kg Zr (소모)
  ZR_DH         = 586 kJ/mol   = 6.42   MJ     / kg Zr
```

**증기가 상한이다.** 노심 동역학을 따로 세우지 않고, 반응속도를 지금 끓는
양(과 증기 재고)으로 자른다. 발열·수소·증기소모는 이 **한 속도**에서 나온다.

```
want   = zr_heat 합 / ZR_DH                     온도가 부르는 속도
avail  = min(boil_rate, (steam_mass-하한)/dt) / ZR_H2O_PER_ZR
rate   = min(want, avail)                        -> zr_heat 도 같이 잘린다
```

상한을 `_fuel_temperature` **한 곳**에만 둔다. `_vessel` 은 안에서 열 번
잘게 돌기 때문에 거기서 다시 걸면 복리로 먹힌다(§5.67).

#### 어디로 가나

기상부 수소는 **밸브 용량**에 질량분율을 곱한 만큼 기존 경로에 실린다.
실현 유량이 아니라 용량인 이유는, 노심이 말라 끓기가 멎어도 열린 밸브는
거기 있는 기체를 내보내기 때문이다.

| 경로 | 종착지 |
|---|---|
| SRV · HPCI/RCIC 구동증기 | 억제수조 → **압력억제실 기상부** |
| 파단 | **드라이웰** |
| 밸브 · 터널 | 복수기 · 2 차 격납 (격납 밖) |

임계유동 질량유량은 분자량의 제곱근에 비례한다 — `H2_CHOKE = 0.335`.

#### 격납용기의 세 번째 기체

```
dw_pressure = dw_p_n2 + dw_p_steam + dw_p_h2
ww_pressure = ww_p_n2 + ww_p_h2 + sat_pressure(수조온도)
R_H2 = 4124 J/kg·K   질소(296.8)의 14 배 — 질량이 작아도 분압은 안 작다
```

다운커머·진공파괴밸브는 섞인 채로 옮기며, 평형 이동량은 **옮기는 기체의
평균 기체상수**로 푼다. 질소로 못박으면 수소를 14 배 과대평가해 평형을
지나치고, 물기둥 때문에 되돌아올 길이 없다.

> **이것이 요점이다.** 수조를 식히면 포화압은 무너지지만 수소 분압은 온도에
> 선형일 뿐이라 거의 그대로다. 그래서 수소가 들어온 뒤에는 수조냉각으로
> 격납압력을 되돌릴 수 없다 — 퍼지로 빼내는 것 말고는 길이 없다.
> 압력억제실에는 직접 배기가 없어 진공파괴밸브로 드라이웰에 넘긴 뒤 뺀다.

표시는 **부피비**다(교범 기준 4 vol%). `H2_ALARM_FRAC = 0.04`.

**다루지 않는 것** — 격납 안 연소(질소 불활성이라 산소가 없다), 원자로건물
기상부, 방사분해 수소, CAD 질소희석, 압력용기 파손.

### 수위의 TAF 아래 가지 — 잠긴 기포만 밀어 올린다

`VESSEL_AREA` 30 m^2 는 **정상수위 근처의 계기 감도**다. 노심 구역까지
균일 단면으로 외삽하면 기하가 안 맞으므로(§5.58), TAF 아래는 남은 재고의
비율로 TAF~BAF 를 잇는다.

기포 팽창은 그 위에 얹히는데, **잠긴 슬라이스의 것만** 센다.

```
frac        = (v_liq + 팽창(frac) - v_min) / (v_taf - v_min)      잠긴 비율
팽창(frac)  = CORE_COOLANT_VOLUME x 평균(축방향보이드 x 잠김) / 100
수위        = TAF + (TAF - BAF) x (frac - 1)
v_min       = MIN_WATER_MASS / rho          바닥 = BAF = 노출 100%
```

잠김은 `submerged_profile(수위)` 가 준다 — 수위와 팽창이 서로를 정하므로
고정점으로 푼다. 기울기가 최대 0.44 인 수축이라 **해는 하나뿐**이고,
재고가 바닥이면 어떤 보이드 분포에서도 BAF 로만 떨어진다. TAF 에서는 위
가지와 정확히 이어진다.

> **왜 노심 평균을 못 쓰나.** `_channel_hydraulics` 는 드러난 노드의
> 보이드를 `VOID_MAX` 로 **강제**한다. 반응도에는 그게 맞다 — 물이 없으면
> 감속이 없다. 그러나 "끓는 물이 얼마나 부풀었나" 를 묻는 자리에서는
> 거짓이다. 그 강제값을 그대로 더하면 물이 빠질수록 팽창이 커져 수위를
> 떠받친다 — 용기를 다 비워도 노출이 69% 에서 멈췄다(§5.66).

같은 이유로 **자연순환도 잠긴 보이드**를 본다.

```
natural_circ = clip( NATURAL_CIRC x sqrt(wet_void / NATURAL_CIRC_VOID),
                     NATURAL_CIRC_MIN, NATURAL_CIRC )
wet_void()   = 잠긴 부분의 평균 보이드 — 노심이 다 잠기면 노심 평균과 같다
```

노심이 덮여 있는 한 두 값이 같으므로 정상 운전·기동·정격 거동은 이 가지와
무관하다. L1~L8 설정치도 전부 TAF **위**다.

### 직류 계열과 디젤 — 계열마다 자기 디젤을 진다

125 VDC 는 네 계열이다. 셋(A1·B1·C1)은 ESF 용이고 각각 비상모선
101·102·103 에서 충전기를 받는다. 넷째(D)는 비안전이고 정상 교류에서 받는다.

**계열별 부하는 짐작이 아니라 표에 있다** — NRC HRTD R-304B Rev 09/11 의
Table 9.4-1~3 (옛 판에서는 이 표가 그림이라 못 읽었다).

| 계열 | 지는 것 |
|---|---|
| A1 (Div I) | RCIC 제어·MOV·보조펌프 · ADS I · 예비스크램 A · **DG1 제어·계자여자·연료유펌프** · 계열 I switchgear 제어 |
| B1 (Div II) | HPCI 제어·MOV·보조펌프 · ADS II · 예비스크램 B · **DG2** · 계열 II switchgear |
| C1 (Div III) | **DG3** · 계열 III switchgear · 안전등급 환기 |
| D | 주터빈 제어·보호(비안전) · 소내 전산기 |

```
dg_dc_ok(i)     = dc_bus_ok(DC_DIV_DG[i])     디젤은 자기 계열 직류가 있어야 뜬다
dc_bus_ok(d)    = (충전기 급전+건전) 또는 축전지 잔량 > 0      (9.4.1 의 두 전원)
turbine_dc_ok   = dc_bus_ok(D)                주터빈 정지 솔레노이드 (fail-safe)
control_power   = 비상모선 하나라도 살아 있음 또는 ESF 축전지 잔량
dc_extra_load() 계열별 추가부하 — HPCI 는 II, RCIC 는 I 에만 걸린다
```

**엔진은 공기로 돈다** (R-304B 9.2.3.2 — 완전용량 기동공기 2 계통, 탱크마다 5 회
기동분). 직류를 잃어도 크랭킹은 되지만 계자를 못 띄우고 출력차단기를 못
닫으므로 모선에는 못 붙는다. 모델은 그 결과만 본다.

> **되돌릴 수 없는 문.** 축전지가 비면 디젤을 못 붙이고, 디젤이 안 붙으면
> 충전기가 급전을 못 받아 축전지가 영영 안 찬다. 전소내정전에서 축전지를
> 다 쓰면 그것으로 끝이다 — 모델이 그 교착을 그대로 낸다.

추가부하 식은 `dc_extra_load()` **한 곳에만** 있다. 방전과 남은시간 표시가
각자 복사본을 갖고 있다가 한쪽만 고쳐지는 사고를 겪었다(§5.65).

### 제어전원은 둘이다

9.4.1 이 125 VDC 를 네 계열로 나눈다. 셋(A1·B1·C1)은 ESF 용이고 넷째(D)는
비안전인데, 원문이 그 안을 이렇게 적는다 — "Two of the systems supply
**control and protective equipment**". 그래서 제어전원이 하나가 아니다.

```
control_power   안전   MSIV 격리 솔레노이드·ADS·SRV 릴리프
                       = 비상모선 하나라도 살아 있거나 ESF 축전지 잔량
turbine_dc_ok   비안전 주터빈 제어·보호 (정지 솔레노이드는 fail-safe)
                       = D 계열 배전모선
dc_bus_ok(d)           충전기(급전+건전) 또는 축전지 잔량 — 9.4.1 의 두 전원
```

D 만 살아 있는 상태는 "제어전원 상실" 이 아니다. 터빈은 제어되고 MSIV 는
닫힌다. 반대로 축전지가 전부 비면 교류가 멀쩡해도 터빈이 선다.

## 3.3b 터빈건물 냉각수 `bop._tb_cooling(dt)` (NRC 11.4 · 11.5)

§3.3 은 압축기가 전원만 있으면 도는 것처럼 썼다. 실기는 냉각수도 있어야
한다 — 11.6.4.1 이 그 연결을 못 박는다. 두 계통이 직렬로 걸린다.

    냉각탑 수조 --[TBSW 개회로]--> 열교환기 --[TBCLCW 폐회로]--> 터빈건물 기기

**서비스수(TBSW, 11.4)** 는 순환수계통과 취수정을 공유하므로(11.4.3.2) 입구
수온이 `cw_cold` 다 — 냉각탑이 더워지면 이 계통도 더워진다. 3 대 중 2 대 운전.
매뉴얼의 발전소는 해수 개회로라 '취수정' 이 바다지만, 이 모델은 §10.4 대로
내륙 냉각탑 부지다. **공유한다는 관계**를 옮긴 것이지 물을 옮긴 것이 아니다.

**폐회로(TBCLCW, 11.5)** 는 한 마디(lumped)로 본다.

```
Q_기기  = TBCLCW_DUTY_BASE + (RATED - BASE) · 부하율
기기출구 = T + Q_기기 / (m_cl · cp)                     ← 토출은 T, 복귀는 이것
Q_HX    = (1 - 우회) · eff · C_min · (기기출구 - 서비스수)  C_min = min(C_sw, C_cl)
C·dT/dt = Q_기기 - Q_HX                                C = V · rho · cp
```

열교환기의 구동 온도차를 **토출(T)이 아니라 복귀(기기출구)** 로 잡는 것이
핵심이다. 토출 35℃ 로는 24℃ 짜리 서비스수에 40 MW 를 못 버린다. 실기가
토출을 35℃ 로 잡으면서도 성립하는 이유가 복귀가 6 K 더 뜨겁다는 것이다.

온도조절밸브(11.5.2.3)는 **위치를 누적**한다 — 그래서 정상편차가 남지 않는다.
실기의 모듈레이팅 밸브가 하는 일과 같다.

```
cmd  = clip(우회 - (T - TBCLCW_SETPOINT) · 0.10, 0, 1)
우회 += clip(cmd - 우회, ±TBCLCW_VALVE_RATE · dt)
```

냉각능력이 모자라면 우회가 0 에 붙어 버티고 그때부터 T 가 뜬다. **0 % 인데
온도가 오르는 것**이 조절 여유가 없다는 신호다.

정격 실측(폐회로만 떼어 6000 초 수렴): 토출 **35.0℃** · 우회 46 % ·
서비스수 +7.0 K. **TBSW 한 대만 남으면** 우회가 0 까지 닫히고 토출이
**36.5℃** 에서 다시 잡힌다 — 열은 그대로 나가되(HX 40 MW) 서비스수
온도상승이 **+14.1 K** 로 두 배가 된다. 정지선(60℃) 근처에도 못 간다.
여유를 잡아먹었을 뿐 아직 계통이 일하고 있다는 뜻이다.

냉각을 통째로 잃으면 온도는 그냥 오른다. 85 psig 폐회로라 실제로는
`TBCLCW_BOIL`(160℃) 부근에서 끓고 그 위는 모델이 없다 — 압축기는 60℃ 에서
이미 섰으므로 결과에는 영향이 없고, 표시가 헛소리가 되는 것만 막는다.

**서지탱크(11.5.2.2)** 가 폐회로를 폐회로답게 만든다. 물이 줄면 여기 수위가
준다 — 매뉴얼이 이 탱크의 목적으로 **누설 검출**을 명시한다. 보충수는
복수탈염수에서 `TBCLCW_MAKEUP_MAX` 까지 자동으로 온다. 그보다 큰 누설이면
결국 비고, 그때 펌프가 흡입을 잃어 유량이 0 이 된다.

**대기펌프는 10 초 뒤에야 밀어낸다** (11.5.4.2). 운전펌프를 잃으면 대기펌프가
기동하지만 토출밸브는 10 초 뒤에 열린다. 그동안 유량이 없다.

### 압축기와의 연결 — 이 계통이 관측 가능한 유일한 통로

```
if 유량 없음      : tb_cool_why = "폐회로 냉각수 상실"
elif T >= 60℃    : tb_cool_why = "냉각수 고온"
else             : tb_cool_why = ""       (식힐 수 있다)
```

`_air` 가 이 판정을 읽는다. 다만 **돌고 있는 압축기만** 과열한다.

```
if 압축기가 돌고 있고 tb_cool_why:  타이머 += dt
   타이머 >= AIR_COOL_TRIP_DELAY -> air_cool_trip 래치
```

전원을 잃어 서 있는 동안에는 걸리지 않는다. 그렇게 두면 LOOP 를 복구할
때마다 운전원이 모르는 래치가 하나 더 생긴다 — §5.48 에서 격리 복귀로 겪은
바로 그 함정이다. 그리고 `reset_air_compressors()` 는 같은 규칙을 따른다:
**신호가 살아 있으면 받지 않고 이유를 말한다.**

압축기 대수는 `air_running` 하나가 센다. 화면이 따로 세면 정지를 모르는
채 "2 대 운전" 이라고 찍는다.

## 3.3c 제어봉 낙하 사고 · RWM (NRC R-304B 7.5)

### 구동장치와 블레이드는 다른 것이다

`rods` 는 **블레이드** 위치, `rod_drive` 는 **구동장치** 위치다. 평소에는
같지만 결합이 풀리면 갈라진다. 낙하 뒤에도 고장난 결합은 자동으로 복구되지 않는다.
패널은 구동/날개를 따로 표시하며 낙하 거리와 제한속도 기준 예상 시간을 제공한다.
전인출된 점검 대상이 없으면 결합 확인을 수행했다고 표시하지 않는다.

```
set_rods(값) :  rod_drive 를 옮긴다
                rods = where(rods_uncoupled, rods, rod_drive)
```

이 한 줄이 사고의 1 번 조건이다. 그리고 그 결과로 **뽑아도 중성자 계측이
안 움직인다** — 실기 운전원이 알아채야 하는 징후(5 번 조건)가 모델에서
그대로 나온다. 실측: 냉간 임계에서 30 초 뒤 출력이

```
가만히 두면 0.0203%  ·  결합 풀린 봉 인출 0.0203%  ·  정상 인출 0.0235%
```

결합이 풀린 인출은 **가만히 있는 것과 똑같다.** 스크램도 못 민다 — CRD
수압은 구동장치를 미는 것이지 블레이드를 직접 밀지 않는다.

### 낙하 에너지 — 적분하지 않고 닫힌 해로

```
ρ  = (k_뽑힘 - k_박힘)/k_뽑힘 × CRDA_ROD_SCALE
ΔT = 2·(ρ - β) / |α_도플러|                       (Fuchs–Nordheim)
h  = h_지금 + UO2_CP·ΔT·첨두 / 4184               [cal/g]
```

폭주는 ms 단위인데 이 모델의 최소 스텝은 0.2 초다. 그 시간축을 새로
만드는 대신 **총 에너지만** 닫힌 해로 얻는다. 계단 삽입 가정이라
보수적이다 — 실제 블레이드는 속도제한기 때문에 3.11 ft/s(0.948 m/s)로
전행정을 3.86 초에 걸쳐 떨어진다(R-304B 2.2.3.4).

에너지는 **떨어진 뒤의 출력 형상대로** 연료에 넣는다. "지속시간이 연료
시정수보다 훨씬 짧아 발생 에너지가 전부 펠릿에 남는다"(7.5.3.1)는 서술
그대로다. 그래서 용융 판정은 기존 `fuel_temp > FUEL_MELT_TEMP` 가 그대로
맡는다 — **주인은 하나다.** 여기서 새로 보는 것은 피복재 천공뿐이다.

| 엔탈피 | 결과 | 모델 |
|---|---|---|
| 170 cal/g | 피복재 천공 문턱 | `clad_damaged` |
| 200~280 | 연료 용융 (**280 이 설계한계**) | 온도 판정이 잡는다 |
| 425 | 용융 완료 | 〃 |

### 격자 보정 — 이걸 안 하면 구분이 사라진다

이 모델은 제어봉을 **60 칸**으로 다루는데 실기는 **137 대**다(2.3.1).
칸 하나가 실기 2.3 대 몫을 쥐고 있으니, 칸을 통째로 뽑은 반응도는
**한 대**가 떨어지는 사고보다 그만큼 크다. 낙하는 한 대가 떨어지는
사건이므로 `CRDA_ROD_SCALE = 60/137 = 0.438` 을 곱한다.

보정 전에는 순서를 지킨 봉도 2.19β 가 되어 즉발초임계였다 — RWM 이 갈라야
할 '준수 vs 위반' 이 아예 구분되지 않았다.

### RWM — 순서를 강제한다

ROD_SEQUENCE 를 12 칸씩 5 그룹으로 자르고, 그룹마다 삽입한계 0 · 인출한계
100 을 준다. 걸린 그룹은 "앞 그룹이 전부 인출한계에 있는가" 로 정한다.

```
인출오류  상위 그룹이 삽입한계 위    -> 하나면 차단
삽입오류  하위 그룹이 인출한계 아래  -> 세 번째에 차단
차단이 걸려도 **오류 봉 자신**은 움직일 수 있다 (되돌리라고 거는 차단이다)
```

구역은 **주증기 유량**으로 가른다: 20% 미만 강제 · 20~30% 표시만 ·
30% 초과 표시도 끔. 키록 우회는 어느 출력에서도 먹는다.

**자동 기동도 이 규칙을 지킨다.** 예전에는 '앞쪽 12 칸' 이라는 창으로만
봐서 앞 그룹이 안 끝났는데 다음 그룹을 건드렸다(실측 인출오류 45 건).
후보를 걸린 그룹 안으로 가두니 0 건이 됐고, 냉간→정격 등반은 그대로 된다.

### 인출 순서는 두 벌이다 — A · B

`ROD_SEQUENCE` 가 하나뿐이면 노심은 늘 같은 자리를 뱅킹한 채 정격에 선다.
실기는 그렇지 않고, 교범이 이유를 그대로 적는다 (R-104B 6.2.2.1 = NRC
Advanced Technology Manual 6.6.4.1, Rev 1210):

> The rods are divided into **two rod groups** which are compatible with the
> Rod Worth Minimizer rod groups. From an all rods full in condition, **the
> operator may choose either of the two groups to begin movement.** Once the
> operator begins to withdraw the first rod in that group, the logic will not
> allow selection of any rods but those in the chosen group, until all rods in
> that group are moved to the full out position.

'두 무리' 가 무엇인지는 RWM 쪽(6.1.2.1.1)이 말한다 — 한 벌을 다 뽑으면
전인출된 봉이 **체커보드(흑백) 무늬**로 남는다. 즉 두 무리는 체커보드의 두
색이고, 모델의 정렬 첫 키가 이미 `(x+y)%2` 였으므로 **예전 순서가 A 이고
반대 색부터 뽑는 것이 B** 다.

```
ROD_SEQUENCES["A"]  (x+y) 짝수 30 대 -> 홀수 30 대     각 색 안에서는 바깥 -> 중심
ROD_SEQUENCES["B"]  (x+y) 홀수 30 대 -> 짝수 30 대     (정렬 규칙은 같다)
```

달라지는 것은 어느 색이 먼저인가 하나뿐인데, 그 하나가 **마지막 RWM 그룹을
통째로 바꾼다.**

```
A 마지막 그룹  41 63 14 36 52 25 32 23 54 45 43 34
B 마지막 그룹  31 13 64 46 22 55 42 53 24 35 33 44      겹치는 봉 0
```

정격에 닿는 순간 덜 뽑혀 있는 것은 언제나 마지막 그룹이므로(자동 인출이
현재 그룹을 끝내고 다음으로 가기 때문에, 7.5.2.1.2), **남는 구역이 통째로
옮겨간다.** §1.2b 의 장전 편차가 못 하던 일이 이것이다.

**고르는 것은 전삽입에서만 된다.** `select_rod_sequence(name)` 이 구동장치
최대 위치를 보고 `RWM_TOLERANCE` 를 넘으면 거절한다 — 교범의 "Once the
operator begins to withdraw the first rod ... will not allow" 그대로다.
바꾸면 걸어 둔 그룹 번호(`_rwm_group`)를 지운다. 새 순서의 그룹 번호이기
때문이다.

원자로가 자기 순서를 들고 있고(`rod_sequence_name`), `rod_sequence` ·
`rwm_groups` 는 거기서 나오는 속성이다. 모듈 상수 `ROD_SEQUENCE` ·
`RWM_GROUPS` 는 **A** 로 남겨 뒀다 — `Reactor()` 의 기본이 A 라 시험과
스냅샷이 늘 같은 기동을 본다.

#### B 가 더 뜨거운 노심이다 — 그리고 그 이유가 FUEL_ZONE 에 있다

같은 장전으로 A 와 B 를 각각 냉간에서 정격까지 자동 기동해 30 시간 평형까지
간 결과:

| | A | B |
|---|---|---|
| 총첨두 | 2.2967 | 2.3031 |
| 반경 첨두 | 1.3659 | **1.5005** |
| MFLPD | 0.9444 | 0.9533 |
| MAPRAT | 0.9999 | 1.0093 |
| MFLCPR | 1.0102 | **1.1033** |

총첨두는 거의 같은데 **반경 첨두와 MFLCPR 이 눈에 띄게 나쁘다.** 원인은
`FUEL_ZONE` 이다. 이 배열은 "반경 연료 존" 이라고 적혀 있지만 A 순서의 평형
출력분포에서 역산해 굳힌 것이라, **A 가 뱅킹해 두는 봉 자리의 그림자 보정**
이 함께 들어가 있다. 매끈한 반경 적합(거리² 의 2 차식)과 비교하면 그대로
드러난다:

```
(3,4) +7245    (4,5) +7087    (4,3) +6195    (5,4) +5987  pcm
```

이 넷이 정확히 A 의 최심 4 대다. A 최심 4 의 잔차 평균은 **+6628 pcm**,
B 최심 4 는 **-3384 pcm** 이다. B 로 돌리면 그 자리의 봉이 뽑혀 있어 보정이
떠 버리고 그 채널이 달아오른다.

**고치지 않았다.** B 전용 존을 같은 방법으로 역산해 봤는데(반복 8 회,
목표는 'A 와 같은 평형 채널출력'), 이득이 분명하지 않았다 — 열적 지표는
좋아졌지만(총첨두 2.03, MAPRAT 0.89) 목표 대비 오차는 오히려 커져
0.035 → 0.20 으로 발산했다. 이득은 있는데 수렴하지 않는 해를 교정 상수로
굳히는 것은 이 저장소가 계속 피해 온 일이다. 그리고 연료는 순서를 모른다 —
장전을 순서마다 따로 두면 그것대로 물리가 아니다.

그래서 **장전은 한 벌, 순서는 두 벌**이다. B 는 여유가 더 좁은 노심이고
그것을 화면이 그대로 보여 준다. MCPR 로 치면 A 1.425 대 B 1.305 로,
둘 다 안전한계 1.07 위다.

#### 정격 직후가 가장 뜨겁다 — 제논 과도 실측

정격(99.5%) 도달을 0 시로 놓고 30 시간 동안 1 시간마다 재 봤다.

```
   h |      총첨두      |      MFLPD      |     MAPRAT      |     MFLCPR
     |    A       B    |    A       B    |    A       B    |    A       B
   0 |  2.490   2.815  |  1.038   1.174  |  1.099   1.243  |  0.899   0.916
   2 |  2.377   2.808  |  0.987   1.149  |  1.045   1.216  |  0.872   0.910
   4 |  2.313   2.672  |  0.948   1.091  |  1.004   1.155  |  0.849   0.931
   6 |  2.195   2.522  |  0.904   1.037  |  0.957   1.097  |  0.841   0.952
  12 |  2.014   2.240  |  0.823   0.927  |  0.871   0.982  |  0.842   0.982
  20 |  2.243   2.271  |  0.926   0.931  |  0.980   0.986  |  0.958   1.033
  28 |  2.314   2.343  |  0.960   0.967  |  1.016   1.024  |  1.019   1.098
```

**둘 다** 정격 직후에 MAPRAT 이 1 을 넘는다 — A 1.099, B 1.243. 제논이 쌓여
형상이 평탄해지며 내려가고, A 는 약 **4 시간**, B 는 약 **12 시간** 만에
1 아래로 들어온다. 12 시간 부근이 가장 평탄하고, 그 뒤 제논 재분포와 추가
인출로 다시 완만히 오른다. 어느 쪽도 스크램하지 않는다.

즉 B 는 **정격 직후 반나절이 A 보다 뚜렷하게 뜨겁고**, 평형에 가서는 총첨두
기준으로 거의 같아진다. 대신 MFLCPR 은 끝까지 1.010 대 1.103 으로 벌어진다.
정격 직후의 이 구간은 A 에서도 원래 한계를 넘던 자리다(§5.25 이래).

> 실기 BPWS 는 시작 무리를 넷(1~4) 중 고르고 짝이 따라온다 — A1/A2/B1/B2 가
> 그것이다. 이 모델은 채널이 60 개라 체커보드 두 색이 곧 두 무리이고, 두 벌이
> 그 축약이다.

### 실측 — RWM 이 있는 이유

냉간 임계(21℃, 전인출 22/60)에서:

| | 가치 (1 대분) | β 대비 | 첨두 | 엔탈피 | 판정 |
|---|---|---|---|---|---|
| 순서대로 다음 봉 | 142 pcm | 0.22 | 3.6 | — | 한계 이내 |
| 건너뛴 중앙봉 | **1336 pcm** | **2.09** | 7.5 | **298 cal/g** | **설계한계 초과** |

가치가 9.4 배 갈린다. 순서가 왜 규칙인지가 숫자로 보인다.

**출력이 오르면 위험이 사라진다.** 같은 중앙봉을 출력별로 재면 2% 에서
이미 0.16β 로 떨어진다 — 보이드가 제어봉 둘레 중성자속을 평탄하게 만들기
때문이다(7.5.3.1). RWM 이 20% 위에서 필요 없는 이유가 그것이다.

## 3.4 제어봉 구동수압계통 `_crd(dt)` (NRC 2.3)

§3 은 스크램이 걸리면 제어봉이 `ROD_INSERT_TIME` 만에 들어간다고 썼다. **무엇이
미는가**를 안 적었다. 제어봉은 중력으로 떨어지지 않는다 — 노심 **아래**에서
물이 밀어 올린다. 미는 것이 없으면 안 들어간다.

### 미는 힘 — 압력이거나 축압기다 (2.3.2.13)

```
scram_force_ok = pressure > CRD_SELF_SCRAM_P    (800 psig g = 5.617 MPa abs)
                 or crd_accum >= CRD_ACCUM_MIN  (0.30)
```

원자로가 고압이면 **원자로압 자신이** 밀어 넣으므로 축압기가 없어도 된다.
축압기가 필요한 것은 **감압된 뒤**다. 둘 다 없으면 못 미는 게 아니라
`CRD_SLOW_FACTOR` (4배) 만큼 **느려진다**.

```
rate = 100 / ROD_INSERT_TIME
if (not scram_force_ok) or sdv_full:
    rate /= CRD_SLOW_FACTOR
rods = max(0, rods - rate·dt)
```

### 구동수 — 어느 모선에서 받나 (Table 9.2-1)

축압기는 CRD 펌프가 충전수 헤더로 채운다(2.3.2.10). 그 펌프가 어느 모선에
실리는지는 Table 9.2-1(Shutdown Board Load List)이 그대로 준다.

```
Bus 101 Red     RHR "A" · Core spray "A" · Service water "A" · CRD water pump "A"
Bus 102 Blue    〃 "B"                                        · CRD water pump "B"
Bus 103 Orange  RHR "C","D" · Service water "C","D"            (CRD 없음)
```

**비상모선**이다. 오래 `bop.power_avail`(외부전원)을 보고 있어서, LOOP 에서
디젤이 펌프를 돌리는 중에도 축압기가 샜다 (CHANGES §5.53).

```
crd_drive_ok = any(bus_ok(CRD_PUMP_BUS[i]) for i in range(crd_pumps))
               CRD_PUMP_BUS = (0, 1)

충전 중이면  crd_accum += dt / CRD_CHARGE_TIME(120 s)     상한 1.0
아니면       crd_accum -= dt / CRD_BLEED_TIME(3600 s)     하한 0
충전 중  = crd_drive_ok
```

결과가 직관과 반대다.

| 상태 | 구동수 | 제어봉 |
|---|---|---|
| LOOP (외부전원만 상실) | 디젤이 받친다 | 움직인다 |
| SBO (디젤까지 상실) | 없다 | **인출·삽입 모두 불가** |
| 101 또는 102 하나 상실 | 남은 펌프 | 움직인다 |
| 101 + 102 상실 | 없다 | 불가 |

`rod_block()` 은 스크램 판정 바로 다음에 이것을 본다. 구동수는 **방향과
무관**하다 — 다른 차단은 인출만 막고 삽입은 허용하지만, 구동수가 없으면
넣지도 못한다. 물이 미는 것이기 때문이다.

`RodAutoControl._move_rods` 도 같은 판정을 쓴다. 전에는 `rod_drive` 를 직접
써서 **모든 인출차단을 우회**했다 — 구동수를 끊어 수동이 막힌 상태에서도
자동은 60 초에 0.75 %p 를 뽑았다.

그래서 **LOOP 만으로는 축압기가 아무 일도 안 난다** — 디젤이 받치고, 게다가
고압이니까. 문제가 되는 조합은 `SBO + 감압`(ADS 뒤, 또는 장기 냉각) 이다.
그 순서를 시험 A8 이 밟는다.

### 배수용기 (SDV) — 밀려난 물이 갈 곳 (2.3.2.11)

제어봉이 들어가면 드라이브 피스톤 **위**의 물이 밀려난다. 그 물을 받는 것이
배수용기다. 평상시엔 공기구동 밸브로 열려 비어 있고, **스크램 신호로 닫힌다.**

```
scrammed        -> sdv_isolated = True          (밸브가 닫힌다)
sdv_isolated    -> sdv_level += (직전 제어봉평균 - 지금 제어봉평균) / 100
아니면 공기 있으면 -> sdv_level -= dt / SDV_DRAIN_TIME(300 s)
```

전 제어봉이 100%→0% 로 들어가면 `sdv_level` 이 **1.0** 늘어난다. 즉 단위는
**스크램 횟수분**이고 `SDV_SIZE = 2.0` 은 두 번 받을 수 있다는 뜻이다.

| 값 | 하는 일 |
|---|---|
| `SDV_SCRAM_LEVEL` 0.85 | 다 차기 **전에** 스크램 (7.3.3.2.7) — 단, `rods.max() > 0.1` 일 때만 |
| `SDV_SIZE` 2.0 | 다 차면 `sdv_full` → 삽입이 4배 느려진다 |
| `sdv_level > 0.05` | `rod_block()` — 배수 전에는 인출 금지 |

`rods.max() > 0.1` 조건이 없으면 **스크램이 자기 자신을 다시 건다.** 스크램이
배수용기를 채우고, 그 수위가 다시 스크램 신호를 내기 때문이다. 이미 다 들어간
뒤에는 걸 이유가 없다.

### 교착에 빠졌던 자리

처음에는 "배수용기가 비어야 RPS 복구" 로 짰다. 그런데 배수 밸브는 **스크램 중에
닫혀 있다.** 배수를 못 해서 복구를 못 하고, 복구를 못 해서 배수를 못 하는
교착이었다. 실기 순서는 반대다.

```
제어봉 전삽입 -> RPS 복구(배수 밸브가 열린다) -> 배수 -> 그제서야 인출 허가
```

그래서 `reset_scram()` 이 `sdv_isolated = False` 로 **배수를 열고**, 인출 차단은
`rod_block()` 이 맡는다. 시험 A8 이 이 한 바퀴를 그대로 돈다.

### 이 모형에 없는 것

- 노치(00–48) 단위 인출·삽입. 제어봉 위치는 연속값 0–100 % 다.
- HCU 137 개 각각. 하나로 묶어 평균만 본다 — 그래서 "제어봉 하나가 안 들어감"
  같은 개별 고장을 못 낸다.
- 드라이브수 헤더 압력·냉각수 유량, CRD 펌프 흡입 여과기 차압.
- 배수용기 수위계 자체의 고장(실기에서 스크램 원인으로 유명한 자리다).

## 3.5 출력 자동제어 `RodAutoControl`

실기 기동 절차를 따르는 자동제어. 제어봉과 재순환 유량을 함께 써서 목표
출력을 좇는다.

**2단계 기동**: 1단계는 최저 유량에서 제어봉을 순서대로 인출하고, 제어봉선
(`ROD_LINE_POWER`)에 닿으면 2단계로 제어봉을 동결하고 유량만 올린다. 실기
BWR 은 출력 상위 30~40% 를 유량으로 조절한다.

**목표 추종**
```
log_error = ln(target / power)
rate = clip(|log_error|·approach, 1/max_period, 1/min_period)
move = (±rate - log_rate)·gain·dt
```

**목표 도달 판정 (히스테리시스)**
```
정지 중이면:  |log_error| > deadband·5  일 때만 다시 움직인다 (약 ±1.5%p)
움직이는 중:  |log_error| < deadband    이면 정지               (약 ±0.5%p)
```
변화율 조건은 쓰지 않는다 — 제논·연소도 때문에 변화율이 완전히 0 이 되지
않아, 목표를 맞춘 뒤에도 제어봉이 끊임없이 떨렸다. 실기 운전원도 ±1~2%
안에서는 손을 뗀다.

**APRM 트립 여유 감시 (부하 감발 시)**
```
margin = (aprm_scram_setpoint - total_power)·100          트립까지 여유 [%p]
유량을 내리려 할 때 margin > APRM_MARGIN_MIN(5%p) 이면 유량 감소
                    아니면 제어봉으로 출력을 먼저 낮춘다
```
APRM 스크램 설정치는 유량에 비례한다(0.66W+51%). 유량 57% 에서 출력은 80%
밖에 안 내려가므로, 유량만 계속 내리면 출력이 트립선에 따라잡힌다. 여유를
지키며 유량과 제어봉을 번갈아 쓰면 스크램 없이 감발할 수 있다.

**자동 해제 조건**: 스크램, 연료 용융·피복재 손상, 붕산 주입(SLC), 터빈 정지.

---

## 4. 비상노심냉각계통 `_eccs(dt)`

### 4.1 자동 기동 신호

```
dw_high = dw_pressure > DW_HIGH_PRESSURE (115.8 kPa)
low2 = water_level < LEVEL_2 (-191 cm)
low1 = water_level < LEVEL_1 (-430.5 cm)
```

**신호는 새로 뜨는 순간에만 기동을 건다**(seal-in). 매 스텝 다시 걸면
운전원이 끈 펌프가 되살아나 수동 조작이 안 먹힌다.
- 새 L2 → HPCI, RCIC 기동
- 새 DW고압 → HPCI, CS 2계열, LPCI 4계열 기동
- 새 L1 → CS, LPCI 기동 + ADS 무장(`ads_armed`)
- `water_level > LEVEL_8`(+49.5) → HPCI·RCIC 하드웨어 트립, CS·LPCI 정지(넘침 방지)

**수위 설정치** (NRC 3.1.3.1, 기기영점 inch → 정상수위 기준 cm). 정상수위
(Level 5) = +37 in 이 0 cm 다.

| 레벨 | 매뉴얼 | 코드 | 하는 일 |
|---|---|---|---|
| L8 | +56.5 in | `LEVEL_8` +49.5 | 주터빈·급수펌프터빈·HPCI/RCIC 트립 |
| L7 | +40.5 in | `LEVEL_7` +8.9 | 고수위 경보 |
| L5 | +37 in | 0 | 정상 수위 |
| L4 | +33.5 in | `LEVEL_4` -8.9 | 저수위 경보 + 재순환 런백(§6.6) |
| L3 | +12.5 in | `SCRAM_LEVEL` -62.0 | 스크램 · ADS 확인신호 |
| L2 | -38 in | `LEVEL_2` -191.0 | HPCI·RCIC 기동 · MSIV 격리 · ATWS-RPT |
| L1 | -132.5 in | `LEVEL_1` -430.5 | 저압 ECCS · ADS 무장 |

**Level 8 트립** (NRC 3.1.3.1.1) — ECCS 자동기동 스위치와 무관한 수위계측
보호기능이라 `eccs_auto` 밖에서 판정한다. 셋을 함께 세운다:
- **주터빈** — 습분이 통째로 넘어가 날개가 깨지는 것을 막는다 → `bop.trip_turbine`
- **급수펌프터빈** — 압력용기를 더 채우지 않게 한다 → `bop.trip_feed_pumps`
- **HPCI·RCIC** — 구동증기관 침수를 막는다 (위 줄)

앞의 둘은 **래치 트립**이라 수위가 내려가도 저절로 안 돌아온다. 운전원이
`bop.reset_feed_pumps()` / `bop.reset_turbine()` 으로 복구한다 — 실기와 같다.

### 4.2 고압계통 (증기터빈 구동 — 전원 무관)

```
ramp = min(1, dt/ECCS_START_DELAY)          ECCS_START_DELAY=30 s
hp_ok = HPCI_P_MIN(0.79) < pressure < HPCI_P_MAX(8.03 MPa)
hpci_flow → HPCI_FLOW(268 kg/s) if (hpci_on and hp_ok) else 0    (1차지연)
rcic_flow → RCIC_FLOW(38 kg/s) if (rcic_on and pressure>1.14 MPa) else 0
```
전동기가 아니라 원자로 증기로 도는 터빈펌프라 전원과 무관. 구동증기 운전범위는
매뉴얼이 **150~1150 psig** 로 준다(NRC 2.7.1 · 10.1.2). 이를 절대압으로 옮겨
`HPCI_P_MAX` = 8.03 MPa, `RCIC_P_MIN` = 1.14 MPa 로 잡았다. HPCI 만은 별도로
**터빈 정지밸브가 닫히는 ~100 psig**(NRC 10.0.7)를 하한으로 쓴다 —
`HPCI_P_MIN` = 0.79 MPa.

### 4.3 저압계통 (전동기 구동 — 전원·모선 필요)

```
cs_n = Σ(cs_on[i] and bus_ok(CS_BUS[i]))       살아있는 CS 계열 수
lp_n = Σ(lpci_on[i] and bus_ok(LPCI_BUS[i]))    살아있는 LPCI 계열 수
cs_flow → CS_FLOW(400)·cs_n·max(0, 1 - p/CS_P_MAX)      CS_P_MAX=1.90 MPa
lpci_flow → LPCI_FLOW(630)·lp_n·max(0, 1 - p/LPCI_P_MAX)
```
원자로 압력이 펌프 체결수두(1.90 MPa)보다 낮아야 주입된다. 계열마다 물린
모선이 다르므로(§6) 모선 하나를 잃으면 그 계열만 멎는다.
계열 수는 매뉴얼 근거가 있다 — CS 는 *"a total of **two** core spray pumps"*
(10.3), LPCI 는 *"The **A and C** pumps … the **B and D** pumps"* (10.4.2.2).

### 4.4 ADS (자동감압) — 전원 의존성 포함

**무장·계수 조건** (NRC 10.2.3.1). 세 가지가 **함께** 서 있어야 타이머가 돈다.

```
lp_running = 살아있는 CS 또는 LPCI 계열이 하나라도 있는가 (모선 포함)
ads_hold = ads_armed
           and water_level < LEVEL_1        # ① 저수위 L1
           and water_level < SCRAM_LEVEL    # ② 저수위 L3 확인신호
           and lp_running                   # ③ 저압 ECCS 펌프 기동 중
if ads_hold: ads_timer += dt
else:        ads_timer = 0.0                # 감쇄가 아니라 리셋이다
ads_open = ads_armed and ads_timer >= ADS_DELAY(105 s) and control_power
```

- **① L1** — HPCI 가 수위를 못 잡았다는 뜻. 정상이면 여기까지 안 온다.
- **② L3 확인신호** — 계기 하나가 튀어 헛작동하는 것을 막는 이중 확인.
- **③ 저압 ECCS 허용신호** — 감압해 놓고 넣을 물이 없으면 오히려 위험하다.
  매뉴얼: *"to ensure that there is reactor vessel inventory makeup available
  prior to initiating the ... time delay."*
- 조건이 하나라도 풀리면 타이머는 **0 으로 리셋**된다(매뉴얼 명시).

> **예전에는 ① 만 보고 무장했고**, 주석에는 근거 없는 '격납용기 고압력'
> 조건이 적혀 있었다(실제로는 코드에 없었다). 매뉴얼을 대조해 보니 격납압력은
> ADS 무장 조건이 아니고, 빠져 있던 것은 **저압 ECCS 펌프 허용신호**였다.

ADS 는 솔레노이드로 공기작동기를 구동해 SRV `ADS_VALVES` 대를 강제 개방하는
릴리프 모드다. **제어전원이 없으면(SBO) 강제 개방을 못 한다** — 그래서
SBO 에서는 자동 감압이 안 일어나고 압력이 safety 모드 SRV 만으로 유지된다.

> **`ADS_DELAY` 는 105 s 로 맞췄다** (예전 120 s). 매뉴얼 10.2.3.1 본문에 5회,
> Figure 10.2-1 에도 "105 SECOND TIME DELAY" 로 나온다.
>
> **`ADS_VALVES` 6 대 · SRV 13 대는 그대로 둔다.** 매뉴얼이 자기 모순이다 —
> 2.5.4.11 은 *"ADS uses **six** of the **thirteen**"* 이라 하지만, SRV 자체를
> 기술하는 2.5.2.1 은 *"The Safety/Relief valves **(11)**"* 이라고 두 번,
> ADS 장(10.2)도 *"**Seven** of the **eleven**"* 이라고 두 번 적는다. 즉
> **매뉴얼의 무게는 11/7 쪽**이다.
>
> **실기 문서가 끝냈다.** 브라운스페리 개선 기술지침서(NRC ML061140182):
>
> > LCO 3.5.1 — "Each ECCS injection/spray subsystem and the Automatic
> > Depressurization System (ADS) function of **six safety/relief valves**
> > shall be OPERABLE."
> >
> > LCO 3.4.3 — "The safety function of **12 S/RVs** shall be OPERABLE."
>
> 12 대 가동요구는 **13 대 설치에 한 대 여유**를 둔 것이다. Table 1.8-1 이 이
> 모델의 3293 MWt BWR/4 를 브라운스페리로 밝히므로 **13/6 이 맞고**, 매뉴얼의
> 11/7 은 더 작은 기준 발전소 값이다(ECCS 유량과 같은 사정 — §규모 보정).
>
> **"계통을 소유한 장을 따른다" 는 기준으로는 틀린 답이 나온다** — 여기서는 소유
> 장(2.5.2.1)이 오히려 11 이라고 말한다. 옳은 질문은 "이 숫자가 어느 발전소인가"
> 이고, 그 답은 매뉴얼 안에 없었다. 매뉴얼만 보면 못 푸는 문제였다(CHANGES §5.59).

### 4.5 수원·억제수조

**수원**: CST 우선, 15% 밑으로 떨어지면 억제수조로 전환(`on_pool`).
```
need = eccs_flow·dt
from_cst / from_pool 로 분배
찬물 유입 시: water_temp = (water_mass·water_temp + need·t_in)/(water_mass + need)
```

**억제수조 가열** (SRV·구동증기 응축):
```
SRV 방출·구동증기 m 유입 시:
  e = pool_mass·CP·pool_temp + m·(h_f + h_fg)      증기 엔탈피 전량
  pool_mass += m ;  pool_temp = e/(pool_mass·CP)
```
증기는 잠열만 주고 사라지는 게 아니라 응축수로 남는다. `POOL_TEMP_MAX` = 170℃
(수치 안전 상한 — 압력억제실 가압 시 포화온도가 올라 100℃ 넘을 수 있음).

### 4.6 정지냉각 (RHR SDC)

```
sw_ok = 서비스수가 흐르는 RHR 루프가 하나라도 있는가 (§6.8)
if sdc_on and pressure < SDC_P_MAX(0.90 MPa) and buses_live > 0 and sw_ok:
    q_max = SDC_DUTY(2%)·RATED_MW·1e6                    용량 상한 65.9 MW
    q_in  = decay_power·RATED_MW·1e6                     지금 물을 데우는 열
    q_net_limit = (SDC_COOLDOWN_LIMIT/3600)·water_mass·CP   순 55℃/h 에 해당
    q_rate = q_in + q_net_limit                          붕괴열을 상쇄하고 남는 몫
    q = min(q_max, q_rate, q_left)
    water_temp -= q·dt/(water_mass·CP)     (하한 AMBIENT_TEMP)
```
압력이 낮아야 투입 가능하고, 열응력 때문에 냉각률이 55℃/h로 제한된다.
전동기 구동이라 `buses_live > 0` 이 필요하고, RHR 열교환기를 쓰므로
서비스수도 있어야 한다.

> **냉각률 제한은 '순 냉각률' 에 건다**: 예전에는 SDC 제열 자체를 55℃/h 상당
> (약 14 MW)으로 묶었다. 그러면 붕괴열이 40~60 MW 나오는 정지 직후에 SDC 가
> 제 힘을 안 써서 물이 거의 안 식었다(실측 -13℃/h). 실기 운전원은 반대로 한다 —
> 붕괴열이 크면 밸브를 더 열어 55℃/h 를 맞추고, 붕괴열이 줄면 조인다. 고친 뒤
> SDC 가 65.9 MW(용량 상한)까지 쓰고 냉각률이 -57~-58℃/h 로 안정됐으며,
> 붕괴열이 줄면서 제열도 52.7 MW 로 자연히 조여든다.

---

## 4.9 증기터널 `_tunnel(dt)` — 격납 **밖** 누설을 보는 눈 (Table 4.4-1)

주증기관 4 개가 격납용기를 뚫고 나와 터빈건물까지 가는 통로가 증기터널이다.
바깥쪽 MSIV 도 이 안에 있다. **여기서 새면 드라이웰 압력이 안 오른다** —
파단 신호(`break_size`)가 못 잡는 자리다. 그 자리를 지키는 것이 터널 온도다.

```
누설    = tunnel_leak · (P/정격) · msiv          [kg/s]
C dT/dt = 누설·(h_증기 - cp·T) - UA·(T - 정상온도)
```

`msiv` 를 곱하는 것이 이 계통의 전부다. 밸브가 닫히면 누설이 끊기고 터널이
식는다 — 표가 적은 목적("**Isolate the Steam Leak**")이 그대로 나온다.
신호가 스스로 원인을 없애는 드문 구조다.

| | 값 | 근거 |
|---|---|---|
| 정상 온도 | 40℃ | NO BASIS |
| 경보 | 70℃ | NO BASIS |
| 격리 | 93.3℃ (200 °F) | NO BASIS |
| 환기 UA | 26 kW/K | NO BASIS |
| 열용량 | 4 MJ/K (시정수 154 s) | NO BASIS |

매뉴얼은 이 신호의 **존재와 목적만** 준다 (Table 4.4-1 한 줄과 §2.5.3.1 의
목록). 숫자는 "누설이 있으면 2 분 안에 걸리고 없으면 안 걸린다" 가 되도록
잡았다. 실측:

```
누설 1.0 kg/s -> 130 초에 93.3℃ 도달, 격리 -> 300 초 뒤 누설 0, 터널 47.7℃
누설 0.4 kg/s -> 77.3℃ 에서 평형 (경보만, 격리 없음)
```

**문턱이 있는 것이 중요하다.** 0.55 kg/s 부근이 경계라, 작은 누설은 환기가
이겨서 경보만 뜬다 — 운전원이 조사할 구간이 생긴다. 문턱이 없으면 모든
누설이 곧바로 격리가 되어 신호가 아니라 스위치가 된다.

누설분은 `steam_out` 에 더해져 압력용기에서도 빠져나간다. 크면 **주증기관
고유량**이 먼저 걸릴 수도 있다 — 같은 사고를 두 각도에서 본다.

## 5. 주증기격리계통 `_isolation(dt)` — 전원 의존성 포함

**격리 신호 판정은 `isolation_signal()` 하나에 있다.** `_isolation` 이 걸 때도,
복귀 버튼이 "받을 수 있나" 를 볼 때도 같은 것을 본다. 전에는 판정이
`_isolation` 안에만 있어 복귀가 신호를 못 보고 래치만 풀었고, 다음 스텝에
그대로 다시 걸렸다 (§5.48).

```
reset_isolation() :
    why = isolation_signal()
    if why:  return False, "격리 신호가 살아 있다 — " + why
    래치 해제 -> True
```

실기의 복귀 스위치도 신호가 살아 있는 동안은 안 먹는다.

> ⚠ **이 문장이 §5.48 당시에는 사실이 아니었다.** 판정을 뺐지만
> `active_isolation_signal()` 이라는 거의 같은 복사본이 남아 제어반이 복귀
> 전에 그쪽을 먼저 불렀고, 그 복사본은 `msiv_auto` 를 안 봤다. 자동격리를
> 꺼 두면 둘이 서로 다른 답을 냈다. §13.20 에서 지웠다 — 같은 사슬에서
> 같은 결함을 **두 번** 찾은 셈이다.


MSIV는 "공기로 열고, 공기/스프링으로 닫는(air to open; air and/or spring to
close)" fail-closed 밸브 (NRC 2.5.2.3).

**격리 신호** (`msiv_auto and not msiv_isolated`일 때, 우선순위 순):
1. **제어전원 상실** (`not control_power`) → "제어전원 상실 (fail-safe)"
2. 저저수위 L2 (`water_level < LEVEL_2`)
3. 주증기관 고방사능 (`clad_damaged`) — 표가 적은 원인이 피복재 총체적 파손이다
4. 주증기관 고유량 (`steam_out > MSIV_HIGH_FLOW·RATED_STEAM_FLOW`, 1.40배)
5. 증기터널 고온 (`tunnel_temp > TUNNEL_ISOL_TEMP` 93.3℃) — §4.9
6. 주증기관 저압력 (`mode_run and pressure < MSIV_LOW_PRESSURE` 5.789 MPa)
7. 복수기 저진공 (`bop.cond_pressure > MSIV_LOW_VACUUM` 30 kPa) — **매뉴얼 밖**

한 번 걸린 자동 격리(`msiv_isolated`)는 운전원이 복귀시켜야 풀린다.
다만 3 번(피복재 손상)은 되돌릴 수 없는 상태라 복귀도 영원히 거절된다.

> **§13.20 이전에는 이 목록이 매뉴얼과 달랐다.** 2·4·6·7 만 있었고 3·5 가
> 없었으며, 매뉴얼 어느 목록에도 없는 드라이웰 고압력이 들어 있었다. 그런데
> 이 문서·코드 주석·VERIFICATION 세 곳이 전부 "표와 맞는다" 고 적고 있었다.
> 지금은 Table 4.4-1 과 §2.5.3.1 **두 독립 목록**과 같다 (수동 포함 여섯).
>
> 7 번은 의도적 이탈(DELIBERATE)이다 — 이 교범은 저진공을 Table 7.3-1 의
> **스크램** 신호로 싣는다. 실기 BWR/4 에서는 격리 신호이기도 해서 남겼다.
>
> 표는 **수치 설정치를 안 준다.** 1.40배·5.789 MPa·30 kPa·93.3℃ 는 전부
> 매뉴얼 근거가 없는 값이다.

**MSIV 위치** (0=전폐, 1=전개):
```
want_open = msiv_demand and not msiv_isolated
if want_open:  msiv += dt/MSIV_OPEN_TIME(25 s)
else:          msiv -= dt/MSIV_CLOSE_TIME(4 s)
```
닫히기 시작(msiv<0.9)하면 터빈이 증기를 잃어 트립. 폐쇄 스크램은 밸브 위치
신호라 압력이 오르기 전에 먼저 걸린다(출력>10%일 때, RUN 모드 근사).

**`control_power` 속성 (제어전원)**: `buses_live > 0 or dc_available`.
- MSIV 솔레노이드·EHC·ADS를 여자하는 전원(비상 AC 모선 + 125V 축전지).
- 정상/LOOP(디젤 급전) → 모선 살아 제어전원 유지 → MSIV 안 닫힘
- SBO(디젤까지 실패) → 교류 모선은 죽지만 축전지가 방전하며 제어전원을 잠시
  더 유지한다(§6.4). 축전지가 고갈되어야 MSIV fail-closed, EHC 정지, ADS 불가.
- 모선은 외부전원 또는 디젤로 급전되므로(§6.2 `bus_ok`), 디젤이 자기 모선을
  받치는 한 LOOP 에서는 제어전원이 유지된다. 디젤까지 실패하고 축전지마저
  바닥나야 완전한 제어전원 상실이다.

---

## 6. 전기계통

### 6.1 소내 교류전원 (2차계통, `bwr4_bop.py`)

**외부전원은 두 갈래다** (NRC 9.2.3.1 / 9.2.3.3).

| | 전압 | 역할 |
|---|---|---|
| 정상원 NSST | 138 kV → 4.16 kV | 평소 모선을 먹인다 |
| 예비원 RSST | 69 kV | 정상원을 잃으면 넘겨받는다 |

`_offsite()` 가 매 스텝 절체를 판정한다.

```
지금 원이 살아 있지 않다면
    정상원이었고 · 예비원이 살아 있고 · fast_transfer_ok  → 예비원으로 (고속절체)
    아니면                                              → 상실 (offsite_source = None)
상실 상태에서는 저절로 붙지 않는다 — restore_offsite() 가 운전원 조작이다
```

절체는 **정상원 → 예비원 한 방향뿐**이다 (매뉴얼 "not in reverse"). 예비원으로
운전 중이면 위 분기에 들어가지 않으므로 이 규칙이 저절로 지켜진다.

`FAST_TRANSFER_TIME` = 5 사이클 = 0.083 s 로 최소 시간간격보다 짧다. 그래서
절체는 한 스텝에 끝나고, 모형이 다루는 것은 '어느 원이 먹이는가' 와 '절체가
성공했는가' 다. 전압 공백은 관성 감쇠 한 번으로 반영한다.

`bop.ac_power` 는 이제 파생 속성이다 (`offsite_source is not None`). 끄고 켜는
기존 코드가 그대로 동작하도록 setter 가 두 원을 함께 조작한다.

```
dip = _offsite()                     절체를 겪었으면 0.083, 아니면 0
target = 1 if ac_power else 0
power_frac += (target - power_frac)·min(1, dt/POWER_COASTDOWN)    POWER_COASTDOWN=4 s
if dip: power_frac ·= exp(-dip/POWER_COASTDOWN)                  절체 중 전압 공백
if power_frac < 0.02: power_frac = 0
```
`power_avail` = `power_frac`. 전동기 구동 대수는 이 비율만큼만:
`circ_running = circ_pumps·power_avail`, `fans_running`, `cond_running` 동일.

### 6.2 4160V 비상모선 3계열

`bus_live` = [True]×3. `BUS_NAMES` = ("101 Red", "102 Blue", "103 Orange").
```
bus_ok(i) = bus_live[i] and (bop.power_avail > 0.5 or dg_running[i])
buses_live = Σ bus_ok(i)
```
모선은 **외부전원 또는 그 모선 전담 디젤** 중 하나라도 있으면 산다. 외부전원이
있으면 디젤은 대기(§6.3). 외부전원을 잃으면 디젤이 받친다 — 이게 LOOP와 SBO를
가른다.

**부하 배분** (NRC Table 9.2-1):
| 모선 | 담당 |
|---|---|
| 101 Red | RHR펌프 A, CS A, 서비스수 A |
| 102 Blue | RHR펌프 B, CS B, 서비스수 B |
| 103 Orange | RHR펌프 C·D, 서비스수 C·D |

```
CS_BUS = (0, 1)              CS A→101, B→102
LPCI_BUS = (0, 1, 2, 2)      LPCI A→101, B→102, C·D→103
RHRSW_BUS = (0, 1, 2, 2)     서비스수도 동일
```
LPCI 4계열이 RHR펌프 A~D와 대응하므로 103(Orange) 하나만 잃어도 LPCI가
4→2계열로 반감. 101+102 상실 시 CS는 0, LPCI만 2계열 잔존.

### 6.3 비상 디젤발전기 3대 `_diesel(dt)`

각 디젤은 4160V 비상모선 하나를 전담한다(101→DG1, 102→DG2, 103→DG3).
step에서 `_ac_power` 직후, `_isolation` 직전에 돈다(모선 급전 상태가 격리
판정에 반영되도록).

**상수**:
| 상수 | 값 | 의미 |
|---|---|---|
| `DG_COUNT` | 3 | 모선당 1대 |
| `DG_RATING_KW` | 3500 kW | 연속정격 (문헌 명시) |
| `DG_START_TIME` | 10 s | 자동기동 신호 후 정격 도달 (문헌 <10초) |
| `DG_FUEL_TANK` | 30000 kg | 연료탱크 (정격 만재 약 35시간) |
| `DG_FUEL_FULL` | 0.24 kg/s | 정격부하 연료소모 |
| `DG_FUEL_IDLE_FRAC` | 0.30 | 무부하 소모 = 정격의 30% |
| `DG_OVERLOAD` | 1.10 | 정격의 110% 초과 시 과부하 |
| `DG_OVERLOAD_TIME` | 5 s | 과부하 지속 이 시간 넘으면 트립 |

**자동기동 신호** (하나면 기동):
```
common_start = dw_pressure > DW_HIGH_PRESSURE or water_level < LEVEL_1   전 디젤
bus_undervoltage = bus_live[i] and not (power_avail > 0.5)               그 디젤만
start_signal = common_start or bus_undervoltage
```
- 공통(드라이웰 고압·L1)은 세 대 다 부른다.
- 모선 전압상실(외부전원 상실)은 그 모선 디젤을 부른다.

**기동 타이밍**:
```
start_signal 이면 dg_starting[i] = True
dg_starting 이면 dg_start_timer += dt
timer >= DG_START_TIME(10s) 이면 dg_running = True (부하 수용 가능)
```
외부전원 상실 후 `power_avail`이 0.5 밑으로 떨어지는 데 코스트다운(약 3.5초)이
걸리고, 그 뒤 10초 → 실측 약 13초에 정격 도달.

**대기 복귀**: `dg_running and offsite and not common_start` → 외부전원이
돌아오면 부하를 벗고 정지(대기)한다.

**부하·용량제한** (외부전원 없이 실제 급전 중일 때만):
```
load = bus_load_kw(i)                             그 모선 안전부하 합 [kW]
dg_load_kw[i] = load
if load > DG_RATING_KW·DG_OVERLOAD (3850 kW):
    dg_overload_timer += dt
    timer >= DG_OVERLOAD_TIME(5s) 이면 → dg_tripped = True (모선 죽음, 수동복구)
else: overload_timer 감소
```

**`bus_load_kw(i)`** — 그 모선에 걸린 부하 합. 모선 하드웨어가 차단돼 있으면
(`bus_live[i]` False) 0 이다.
```
상시부하   BUS_HOUSE_KW(150) = 충전기 50 + 480V MCC·조명·계장 100
                              (모선이 살아 있으면 항상 — NRC 9.2.2.1/9.4.1)
디젤보조   DG_AUX_KW(100)     (그 모선의 디젤이 돌 때만 — NRC 9.2.2.2:
                              재킷수·윤활유 펌프, 환기팬)
CS 펌프    CS_PUMP_KW(900)   × (cs_on 이고 CS_BUS[k]==i 인 계열 수)
LPCI 펌프  LPCI_PUMP_KW(1400) × (lpci_on 이고 LPCI_BUS[k]==i)
서비스수   RBSW_PUMP_KW(454) × (rbsw_on 이고 RHRSW_BUS[k]==i 인 펌프 수)
드라이웰   DW_COOLER_KW(300)  (dw_coolers 켜져 있으면)
```
상시부하가 없으면 큰 펌프를 다 세웠을 때 부하가 정확히 0 이 되어, 발전기가
돌면서 아무것도 안 먹이는 그림이 나온다 (§5.36 에서 고쳤다).

| | Bus 101 | Bus 102 | Bus 103 |
|---|---|---|---|
| 정상 (외부전원) | 904 | 904 | 450 |
| LOOP (디젤 급전) | 704 | 704 | 250 |
| 안전계통 전부 기동 | 3304 (94%) | 3304 (94%) | 4258 (122%) |

용량 제한의 핵심: **재순환펌프(약 6 MW/대) > 디젤 3500 kW**. 그래서 디젤로는
재순환을 못 돌리고, 정전 시 노심유량은 자연순환분만 남는다. 정상 배분에선
101·102 가 트립선(3850 kW) 안에 들고, 103 만 RHR 두 대를 지고 있어 전 계통을
동시에 기동하면 넘는다.

**연료 소모** (기동 중·운전 중):
```
frac = dg_load_kw[i] / DG_RATING_KW               부하율
burn = DG_FUEL_FULL·(DG_FUEL_IDLE_FRAC + (1-DG_FUEL_IDLE_FRAC)·frac)
dg_fuel[i] -= burn·dt  (0 이면 정지)
```
정격 만재 약 35시간, 무부하 대기 약 116시간.

**파생/조작**: `diesels_running` = 가동 대수. `reset_diesel(i)` = 과부하 트립
복구(운전원). 정비 차단은 `dg_enabled[i] = False`(기동 안 함).

### 6.4 125V 직류전원 (축전지) `_battery(dt)`

교류가 다 죽어도 축전지가 제어전원을 잠시 더 유지한다. 이 여유분이 SBO 에서
RCIC 제어를 잃기까지의 시간(후쿠시마 시나리오)을 만든다. step 에서 `_diesel`
직후·`_isolation` 직전에 돈다(그 순간 RCIC 유량이 정해진 뒤 방전량을 계산).

**4 계열이다** (NRC 9.4.1). 교류를 3계열로 나눈 것과 같은 사상이다.

| 계열 | 성격 | 충전기 급전원 |
|---|---|---|
| A1 | ESF | Bus 101 |
| B1 | ESF | Bus 102 |
| C1 | ESF | Bus 103 |
| D | 비안전 | 정상(외부) 전원 |

계열마다 전원이 둘이다 — **충전기 1 + 축전지 1**. 안전계열 충전기는 비상
교류모선에서, 비안전은 정상 전원에서 받는다(9.4.3.4). **그래서 직류 계열이
교류 계열의 운명을 물려받는다** — LOOP 에서는 비안전 계열만 방전한다.

**상수**:
| 상수 | 값 | 의미 |
|---|---|---|
| `DC_DIVISIONS` | 4 | ESF 3 + 비안전 1 (9.4.1) |
| `DC_ENDURANCE_H` | 8.0 h | 상시부하 기준 지속시간 — **근거 약함**, 아래 참조 |
| `DC_RECHARGE_H` | 24.0 h | 최저전압 → 만충 (9.4.3.1) |
| `DC_DRAIN_IDLE` | 1.0 | 상시 제어·계측 부하 (기준 배율) |
| `DC_DRAIN_HPCI` | 1.6 | HPCI 운전 중 배율 |
| `DC_DRAIN_RCIC` | 1.3 | RCIC 운전 중 배율 |

**상태**: `dc_div_charge[4]`, `dc_charger_ok[4]`, `dc_available`(안전계열 중
하나라도 살아 있는가). `dc_charge` 는 파생 속성으로 **안전계열 최소값**이다.

**충전/방전** (계열마다):
```
fed = dc_charger_ok[d] and dc_charger_fed(d)    # 9.4.3.4 의 급전원
if fed:  charge += dt/(DC_RECHARGE_H·3600)
else:
    load = 1/DC_DIVISIONS                       # 상시부하 몫
    if ESF: load += (HPCI·RCIC 추가분)/살아있는 ESF 계열 수
    charge -= (load/(1/DC_DIVISIONS))·dt/(DC_ENDURANCE_H·3600)
```
상시부하만이면 배율이 1.0 이라 계열 하나가 8h 간다 — 스칼라 시절과 같다.
`dc_hours_left` 는 **안전계열 중 가장 오래 버티는 것**(= 제어전원을 잃는 시각).

> **`DC_ENDURANCE_H` 8h 는 매뉴얼값이 아니다.** 9.4.3.3 은 *"worst case loads
> for two hours"* 라고 하는데, 그 최악은 LOCA 와 전원상실이 겹쳐 첫 1분간 ESF
> 기동 돌입부하가 걸리는 경우다. 모델의 8h 는 **상시부하** 기준이라 기준이
> 다르다. 다만 모델 최대 부하(HPCI+RCIC)로도 4.2h 라 2h 에 못 미치고, 차이가
> 곧 돌입부하다 — 방전식에 그 항이 없다. 돌입부하를 넣지 않은 채 8 을 2 로
> 바꾸면 상시부하 지속시간이 4배 짧아지므로 값은 그대로 두었다.

**제어전원과의 연결**: `control_power = buses_live > 0 or dc_available`.
교류가 다 죽어도 축전지가 살아있으면 제어전원이 유지된다 — MSIV 가 닫히지
않고, EHC·ADS 도 (전원 측면에서는) 살아 있다.

**HPCI·RCIC 와의 연결** (`_eccs`): 증기로 돌지만 제어·밸브 구동에 직류가
필요하다. `dc_available`이 False 면 조절 불능이 되어 유량이 0 으로 간다.
```
tgt = HPCI_FLOW if (hpci_on and hp_ok and dc_available) else 0
tgt = RCIC_FLOW if (rcic_on and p>RCIC_P_MIN and dc_available) else 0
```

**SBO 거동** (디젤까지 실패, 실측): 축전지 8h 방전 → 약 7.7h 에 직류 상실로
RCIC 정지 → 약 9.0h 에 노심 노출 → 약 9.9h 에 피복재 손상. 후쿠시마 1~3호기
타임라인과 맞는다. RCIC 는 그 전까지 수위가 L2 로 내려갈 때마다 자동기동해
L8 까지 채우고 멈추기를 반복하며 노심을 지킨다.

### 6.5a 재순환 지령 → 실제 속도 `_recirc(dt)`

지령 `recirc_demand_a/b`와 실제 속도 `recirc_a/b`를 분리하며, 실제 속도는
이 함수 한 곳에서만 갱신한다. A/B 각각에 같은 규칙을 독립적으로 적용한다.

```
정상: target = clip(demand, 0, RECIRC_MAX)
런백: target = clip(demand, 0, RECIRC_RUNBACK)
RPT 또는 AC 상실: target = 0, demand = 0
speed(t+dt) = target + (speed(t)-target)*exp(-dt/tau)
```

정상 추종 `RECIRC_TAU=5 s`, 트립/런백 `RPT_COASTDOWN=5 s`는 교육용 관성
근사다. 5초 만에 목표에 도달한다는 뜻이 아니라 차이의 약 63%를 이동한다.
`RECIRC_RATE`는 자동 운전원이 지령을 바꾸는 속도이며 펌프 물리 시정수와 다르다.
정상 최저 운전속도 30%와 정지 지령 0%를 구분한다. 노심유량은 실제 펌프속도에
보이드 의존 자연순환을 결합하므로 펌프가 정지해도 노심유량은 0이 아닐 수 있다.

### 6.5 재순환 펌프 전원 의존성 `_ac_power(dt)`

전원 상실 시 지령을 지운다. `_recirc`가 0으로 관성 감속하며, RPT도 동시에
걸렸다고 같은 스텝에서 감속을 두 번 적용하지 않는다. 복전만으로 자동 재기동하지
않는다. 운전원이 새 지령을 주어야 한다. 구형 지령 없는 저장상태의 복원은
기존 실제 속도를 지령으로 사용한다.

### 6.6 재순환 펌프 자동 트립 `_recirc_trip(dt)`

NRC R-104B §7.2.3.2는 전동기/구동장치 차단기를 여는 트립을 설명한다.
따라서 RPT 이후 실제 펌프속도는 최저 운전속도 30%가 아닌 **0%**로 내려간다.
과거 §5.37의 '0으로 감속하는 함수와 30%로 끌어올리는 함수의 충돌' 수정만으로는
AC가 살아 있는 RPT가 해결되지 않았다. 현재는 지령·트립 신호와 실제 속도의
갱신 역할을 나누어 두 경우를 함께 교정했다.

- EOC-RPT: 터빈 트립/부하상실, `total_power > RPT_POWER(0.30)`.
- ATWS-RPT: 스크램하지 않은 고압력 또는 L2 저저수위.
- L4 런백: L4 저수위 + 실제 급수펌프 상실; `RECIRC_RUNBACK`까지 운전속도 제한.
- RPT가 런백보다 우선하며 신호가 살아 있으면 `reset_rpt()`가 복귀를 거절한다.
- RPT 복귀는 래치만 해제하고 지령은 0을 유지한다. 별도 재기동 지령이 필요하다.

### 6.7 소내부하·발전 (`bop.step`)

```
aux_mw = AUX_RECIRC·recirc³ + AUX_CIRC_EACH·circ_running + AUX_FAN_EACH·fans_running
       + AUX_CONDPUMP_EACH·cond_running + (AUX_HOUSE if ac_power else 0)
       (12·recirc³)  (1.51/대)           (0.25/대)         (3.0/대)        (3.0)
net_mwe = gross_mwe - aux_mw
```
재순환 부하는 속도의 세제곱(펌프 법칙).

`AUX_CIRC_EACH` = 1.51 MW 는 NRC 11.1.2.1 의 "Each motor is rated for 1,500 hp"
(= 1.12 MW)에 **규모 보정 ×1.35** 를 적용한 값이다 — 11장은 2436 MWt 급 발전소를
기술하고 이 모델은 3293 MWt 다(§규모 보정). 예전 2.5 MW 는 근거가 없었다.
나머지 소내부하(`AUX_RECIRC` 12 MW · `AUX_CONDPUMP_EACH` 3 MW · `AUX_HOUSE` 3 MW)는
매뉴얼에 마력이 없어 대표값이다.
정격에서 총 소내부하 약 26 MW, 순발전 약 1073 MWe (실기 1065~1093).

### 6.8 원자로건물 서비스수계통 (RBSW) `_rbsw(dt)`

최종 열침원으로 가는 개회로다. 잔열제거 열교환기의 관측(tube side)을 이 물이
지나므로, **여기가 막히면 전원이 멀쩡해도 RHR 이 열을 못 버린다.** step 에서
`_diesel` 직후에 돈다.

**상수** (NRC 11.2.2):
| 상수 | 값 | 의미 |
|---|---|---|
| `RBSW_PUMPS` | 4 | 펌프 A~D |
| `RBSW_PUMP_FLOW` | 734 kg/s | 펌프 1대 (문헌 8600 gpm = 543, ×1.35 규모보정) |
| `RBSW_PUMP_KW` | 454 kW | 펌프 1대 (문헌 450 Hp = 336, ×1.35 규모보정) |
| `RBSW_LOOP` | (0,1,0,1) | A·C→루프1, B·D→루프2 |
| `RBSW_INTAKE_TEMP` | 20 ℃ | 취수정 수온 (최종 열침원, 계절 가변) |
| `RBSW_HX_RISE` | 8 K | 열교환기 통과 시 서비스수 승온 |
| `DG_COOLING_LOOP` | (0,1,0) | DG1·DG3→루프1, DG2→루프2 |
| `DG_OVERHEAT_TIME` | 300 s | 냉각수 없이 디젤이 버티는 시간 |

펌프 전원은 `RHRSW_BUS = (0,1,2,2)` — 4160V 비상모선에서 딴다. 정상 운전에서는
루프당 1대(A, B)만 돌린다(문헌의 "A또는C / B또는D").

**루프 유량**:
```
for k in 0..3:
    if rbsw_on[k] and bus_ok(RHRSW_BUS[k]):  flow[RBSW_LOOP[k]] += RBSW_PUMP_FLOW
if not rbsw_isolated:                        헤더가 열려 있으면 공통 헤더로 모여
    flow = [total/2, total/2]                두 루프가 나눠 쓴다
rhrsw_temp = max(RHRSW_MIN_TEMP, rbsw_intake_temp)
```

**헤더 격리** (NRC 11.2.1): `dw_pressure > DW_HIGH_PRESSURE`(LOCA) 또는
`buses_live < N_BUS`(전압상실)이면 토출 헤더 격리밸브가 닫혀 **두 독립 루프로
갈라진다.** 갈라지면 한 루프의 펌프가 다 죽어도 다른 쪽은 살지만, 루프 사이로
물을 돌려쓸 수 없다. 그래서 실기는 루프마다 펌프를 나눠 둔다.

**RHR 과의 연결** (§8): 열교환기는 자기 루프에 서비스수가 흘러야 쓸 수 있고
(`usable = min(RHR_LOOPS, buses_live, sw_loops)`), 제열량은 전열면 능력과
서비스수 수송한계 중 **작은 쪽**이다:
```
q_ua = n_hx · RHR_HX_UA · (pool_temp - sw)      전열면 능력
q_sw = Σflow · CP_WATER · RBSW_HX_RISE          서비스수가 실어 나를 수 있는 열
q_hx = min(q_ua, q_sw)
```
정지냉각(SDC)도 같은 열교환기를 쓰므로 서비스수가 없으면 작동하지 않는다.

**디젤과의 상호의존** (NRC 11.2.3.2): RBSW 는 비상 디젤 엔진도 식힌다. 즉
**디젤은 서비스수 펌프에 전기를 주고, 서비스수는 디젤을 식힌다.** 냉각수가
끊긴 채 `DG_OVERHEAT_TIME`(300초)이 지나면 그 디젤은 과열 트립되어 운전원이
복구해야 한다. 서비스수를 다 잃으면 디젤도 따라 죽는 연쇄가 생긴다 — 실측으로
서비스수 전정지 후 301초에 디젤 3대가 모두 멎고 모선이 0/3 이 된다.

**엔진 냉각 타이머**: 열은 정지한다고 사라지지 않는다 — 냉각수가 돌아와야
식는다.
```
if 그 루프에 서비스수가 흐르면:  dg_cool_timer -= dt·2      (식는다)
elif 디젤이 운전 중이면:          dg_cool_timer += dt        (달아오른다)
                                 >= DG_OVERHEAT_TIME 이면 과열 트립
```
`reset_diesel(i)` 은 `dg_cool_timer > 0` 이면 복구를 거부하고 사유를 돌려준다.
(예전에는 디젤이 멎으면 타이머를 0 으로 되돌려, 복구 버튼을 누를 때마다
'차가운 엔진' 으로 되살아났다. 그걸 반복하면 냉각수가 없는데도 축전지가
영원히 충전되는 모순이 생겼다.)

---

## 7. 격납용기 (Mark I) — `Containment` 클래스 / `_containment(dt)`

드라이웰 기상부와 압력억제실(토러스) 기상부를 각각 **질소 + 증기 분압**으로 다룬다.

### 7.1 상수

| 상수 | 값 | 의미 |
|---|---|---|
| `DRYWELL_VOLUME` | 4500 m³ | 드라이웰 자유체적 (159,000 ft³ = 4502) |
| `WETWELL_GAS_VOLUME` | 3400 m³ | 압력억제실 기상부 (119,000 ft³ = 3370) |
| `POOL_VOLUME` | 3822.8 m³ | 억제수조 물 (135,000 ft³, Table 4.1-1) |
| `CONT_DESIGN_PRESSURE` | 528.8 kPa | 설계압 (62 psig, Table 4.1-1) |
| `CONT_NORMAL_PRESSURE` | 108 kPa | 정상 봉입압 (약 1 psig) |
| `DW_HIGH_PRESSURE` | 115.8 kPa | 고압력 신호 (2 psig) — 스크램·ECCS·격리 |
| `DW_COOLER_DUTY` | 2.5 MW | 드라이웰 냉각기 용량 |
| `DW_GAS_CAP` | 2.0e7 J/K | 기상부+내부소물 열용량 |
| `DW_WALL_CAP` | 1.8e8 J/K | 강재·구조물 열용량 |
| `DW_WALL_UA` | 1.5e5 W/K | 기상부↔강재 열전달 |
| `DOWNCOMER_SUBMERGENCE` | 1.2 m | 다운커머 잠김깊이 |
| `VACUUM_BREAKER_DP` | 3.4 kPa | 진공파괴밸브 작동 차압 (0.5 psi, 4.1.2.3) |
| `VENT_K` | 60 kg/s/kPa | 다운커머 유동계수 |
| `R_N2` | 296.8 J/kg·K | 질소 기체상수 |

> Table 4.1-1 은 **외압 설계값 2 psig** 도 준다(드라이웰·압력억제실 공통).
> 모델은 내압만 판정하므로 이 한계는 표현되지 않는다 — 진공파괴밸브가
> 그 상황을 막아 주긴 하지만, 설계값 자체는 코드에 대응물이 없다.

### 7.2 압력 (분압 합)

```
dw_p_n2 = dw_n2·R_N2·(dw_temp+273.15)/DRYWELL_VOLUME       질소 분압
dw_p_steam = min(증기분압, sat_pressure(dw_temp))          증기 분압(포화압 상한)
dw_pressure = dw_p_n2 + dw_p_steam

ww_pressure(pool_temp) = ww_p_n2 + sat_pressure(pool_temp)   수조 표면과 평형
```
**핵심**: 압력억제실 압력은 억제수조 수온에 묶여 있다. 수조가 데워지면 표면
포화압이 올라 격납 압력이 따라 오른다. 그래서 RHR로 수조를 식히지 않으면
결국 설계압에 닿는다.

> 설계압(528.8 kPa)에 닿는 수조 온도는 약 142℃ 인데, 같은 표가 **설계온도를
> 281°F(138.3℃)** 로 준다. 실기에서는 **압력보다 온도가 먼저** 한계에 닿는다.
> 모델은 압력만 판정하므로, 수조가 138℃ 를 넘으면 압력이 아직 설계압 아래여도
> 이미 설계조건 밖으로 읽어야 한다.

이 온도들은 손으로 적지 않는다. `pool_temp_at(p)` 가
`p = 질소분압(T) + 포화압(T)` 를 이분법으로 풀어 준다.

```
POOL_TEMP_DESIGN = pool_temp_at(CONT_DESIGN_PRESSURE)  = 141.7 C
POOL_TEMP_ALARM  = pool_temp_at(CONT_ALARM_PRESSURE)   = 103.5 C
```

설계압을 고치면 온도가 저절로 따라온다. 예전에는 도움말이 짝인 온도를
손으로 들고 있어, 설계압을 487 → 528.8 kPa 로 고쳤을 때 138℃ 가 남았다
(CHANGES §5.27).

### 7.3 열수지 `Containment.step(dt, pool_temp, q_in, steam_in, h_in)`

**드라이웰 냉각기** (비례제어): `duty = DW_COOLER_DUTY·clip((dw_temp-DW_TEMP0)/8 + 0.4)`.
전원 없으면(`buses_live == 0`) 정지.

**기상부·강재 2노드 열전달** (기상부는 지수해로 적분 — 큰 dt 안정):
```
q_break = steam_in·(h_in - CP_WATER·dw_temp)      유입증기가 식으며 내놓는 열
q_net = q_in + q_break - duty
tau = DW_GAS_CAP/DW_WALL_UA
dw_temp = t_eq + (dw_temp - t_eq)·exp(-dt/tau)     t_eq = dw_wall + q_net/DW_WALL_UA
dw_wall += DW_WALL_UA·(dw_temp - dw_wall)·dt/DW_WALL_CAP
```
강재는 시정수 약 20분으로 뒤따라 식으므로, 살수로 압력은 몇 분에 떨어져도
벽은 한참 뜨겁다 — 실기 거동.

**다운커머 (드라이웰 → 압력억제실)**:
```
leg = 995·9.81·DOWNCOMER_SUBMERGENCE               물기둥 차압 (약 11.7 kPa)
dp = dw_pressure - ww_p - leg
if dp > 0:
    m_eq = dp/(a_d + a_w)                          목표(차압=물기둥)까지의 이동량
    m = min(총가스·0.5, m_eq, VENT_K·dp/1e3·dt)
    질소·증기를 비율대로 압력억제실로 이송, 증기는 수조에서 응축(잠열→수조)
```
`m_eq`로 한 스텝에 목표를 지나치지 않게 한다 — 지나치면 물기둥 때문에
되돌아올 경로가 없어 압력이 어긋난 채 고착된다.

**진공파괴밸브 (압력억제실 → 드라이웰)**: `ww_p > dw_pressure + VACUUM_BREAKER_DP`
이면 질소가 역류. 드라이웰 바닥 응축수 배수와는 **별개 경로**다 — 예전에는
`elif` 로 응축수 조건 뒤에 붙어 있어서, 드라이웰에 물이 맺히는 스텝마다
역류 계산이 통째로 건너뛰어졌다(사고+살수처럼 둘 다 해당하는 상황).
질소를 다 넘겨도 수조 포화압 때문에 여전히 높으면 증기
자체가 드라이웰로 넘어가 응축되며 그 잠열만큼 수조가 식는다(격납살수가
수조까지 식히는 실제 경로).

**드라이웰 → 압력용기 열손실**: `_vessel_step`의 `AMBIENT_LOSS·(water_temp - dw_temp)`가
이 열의 출처. 드라이웰 냉각기가 멎으면 드라이웰이 데워지고 이 열손실도 준다.

### 7.4 격납용기 퍼지 `_purge(dt)`

```
if dw_purge and dw_pressure > PURGE_FLOOR(106 kPa):
    m = min(PURGE_RATE(6 kg/s)·dt, dw_n2·0.02)
    dw_n2 -= m
```
SRV를 오래 쓰면 압력억제실 질소가 진공파괴밸브로 드라이웰로 넘어가고,
수조를 식혀도 물기둥 때문에 돌아오지 않아 드라이웰 고압력이 안 풀린다.
퍼지로 질소를 대기방출계통으로 빼낸다(방사성 가스 방출이라 운전원이 직접 조작).

---

## 8. 잔열제거계통 격납모드 `_rhr(dt)`

열교환기 2대(`RHR_LOOPS`), 관측쪽 서비스수·쉘쪽 RHR물. 어느 모드든 펌프
유량은 자기 열교환기를 지난다.

```
rhrsw_temp = max(RHRSW_MIN_TEMP(10), rbsw_intake_temp)  최종 열침원 = 서비스수(§6.8)
sw_loops = 서비스수가 흐르는 루프 수                     rbsw_loop_ok(lp)
usable = min(RHR_LOOPS, buses_live, sw_loops)          모선·서비스수 둘 다 필요
free = usable - (1 if sdc_on else 0)                   정지냉각이 열교환기 하나 가져감
spc = clip(spc_loops, 0, free)                         수조냉각 계열
spray = clip(spray_loops, 0, free - spc)               격납살수 계열
```

**수조냉각 (열교환기 공용)**:
```
n_hx = spc + spray
q_ua = n_hx·RHR_HX_UA·max(0, pool_temp - sw)            전열면 능력 (295 kW/K)
q_sw = Σrbsw_flow·CP_WATER·RBSW_HX_RISE                 서비스수 수송한계 (§6.8)
q_hx = min(q_ua, q_sw)                                  둘 중 작은 쪽이 실제 제열
pool_temp -= min(q_hx·dt/cap, pool_temp - sw)
spc_duty = q_hx/1e6 [MW]
```

**격납살수** (`spray > 0`): 열교환기를 지난 살수(`t_spray`)가 드라이웰
기상부를 식히고, 드라이웰 증기를 응축시켜 수조로 떨어뜨린다. 응축수는 현열만
들고 간다(잠열은 증기가 드라이웰 유입 시 이미 계상).

RHR 펌프·서비스수 펌프 모두 전동기 구동 → 전원 없으면 어느 모드도 못 씀.
정전이 길어지면 격납 열을 버릴 방법이 사라진다.

---

## 9. 고농도붕산주입계통 (SLC) `_slc(dt)`

### 9.1 상수

| 상수 | 값 | 의미 |
|---|---|---|
| `SLC_TANK_VOLUME` | 18.36 m³ | 저장탱크 (4850 gal) |
| `SLC_SOLUTION_RHO` | 1100 kg/m³ | 오붕산나트륨 13wt% 용액 밀도 |
| `SLC_BORON_FRACTION` | 0.0238 | 용액 중 붕소 질량분율 |
| `SLC_PUMP_FLOW` | 2.7 kg/s | 펌프 1대 (39 gpm) |
| `SLC_PUMPS` | 2 | 100% 용량 펌프 |
| `SLC_MIX_TAU` | 120 s | 노심까지 섞이는 시정수 |
| `BORON_WORTH` | -10 pcm/ppm | 붕소 반응도가 (고온) |

`SLC_BORON_FRACTION` 근거: 오붕산나트륨 10수화물 붕소분율 0.183 × 13wt% = 0.0238.

### 9.2 계산

**기동** `arm_slc(pumps)`: `slc_armed=True`, `slc_valves_fired=True`(불가역),
`slc_pumps_on=pumps`, RWCU 자동격리.

**주입**:
```
m = min(SLC_PUMP_FLOW·slc_pumps_on·dt, slc_tank)
slc_tank -= m ;  slc_flow = m/dt
boron_mass += m·SLC_BORON_FRACTION       압력용기 안 붕소
water_mass += m                          용액 자체도 물
```

**손실** (붕소는 증기로 안 날아감, 넘침·RWCU로만):
```
loss = overflow + (rwcu_flow if not slc_armed else 0)
boron_mass -= boron_mass/water_mass·loss·dt
```

**노심 농도** (바닥 주입 → 제트펌프 난류로 섞임, 시간 지연):
```
boron_ppm = boron_mass/water_mass·1e6            압력용기 평균 농도
boron_core += (boron_ppm - boron_core)·min(1, dt/SLC_MIX_TAU)   노심이 느끼는 농도
```
`boron_core`가 `_reactivity`의 `BORON_WORTH·boron_core` 항으로 들어간다.
탱크를 다 넣으면 약 2400 ppm → 약 -24000 pcm(제어봉 전량가치와 같은 자리수).
출력 0까지 1~2시간 — 스크램 대체품이 아닌 느린 최후수단.

### 9.3 RWCU 취출

`RWCU_BLOWDOWN_MAX` = 3.2 kg/s (≈ 4분에 1인치). 증기 배출 외에 물을 줄이는
유일한 정상 경로. SLC 기동 시 자동 0(붕소를 도로 걷어내지 않게).

---

## 10. 2차계통 (Balance of Plant) `bwr4_bop.py`

### 10.1 주증기 분배 (`bop.step`)

```
avail = steam_turbine + steam_bypass
sjae_steam = min(SJAE_STEAM(4), avail·0.5) if (sjae_on and reactor_p>700 kPa) else 0
throttle_flow = steam_turbine·(1 - sjae_steam/avail)      가감밸브 통과
bypass_flow = steam_bypass·(...)
터빈 정지면 throttle 전량이 bypass로
```

### 10.2 터빈 팽창선·발전

```
s_in = s_g(sat_temp)                              입구 엔트로피 (포화증기)
tc = max(cond_temp, t_sat(EXHAUST_FLOOR))         배기온도 (초킹 한계 3.4 kPa)
x_is = (s_in - s_f(tc))/(s_g(tc) - s_f(tc))       등엔트로피 배기 건도
h_is = h_f(tc) + x_is·h_fg(tc)
dh = TURBINE_EFF·(h_in - h_is)                    실제 엔탈피 낙차 (효율 0.82)
work = dh·(exhaust_flow + Σ(추기·HEATER_PHI))     추기점 팽창률 반영
gross_mwe = (work/1000 - pump_mw)·GENERATOR_EFF   (0.985)
```

### 10.3 급수가열기 `_heater_train` (드레인쿨러 1 + 추기 5단)

```
HEATER_SHELL_T = (84, 119, 153, 187, 218.6) ℃     정격 추기 포화온도
shell[i] = t_sat(max(cond_pressure, HEATER_P[i]·load))    부하 비례 추기압
각 단 출구온도 = shell[i] - TTD[i]
추기량은 급수쪽 열수지 + 드레인 캐스케이드로 산정
```

**말단온도차는 고정이 아니라 유량에 따라 변한다.** 응축측이 등온이므로
유용도는 `ε = 1 - exp(-NTU)` 이고, 따라서

```
TTD[i] = (shell[i] - t_in[i]) · exp(-NTU[i])
NTU[i] = HEATER_NTU[i] · (계열수/FWH_STRINGS) · (RATED_STEAM/m_fw)
```

`HEATER_NTU` 는 `_design_ntu()` 가 **설계점에서 TTD 가 `HEATER_TTD`(3℃) 가
되도록 역산**한 값이다 (`_design_phi` 와 같은 방침 — 손으로 넣지 않아 단
구성이나 TTD 를 바꿔도 저절로 따라온다).

| 단 | LP1 | LP2 | LP3 | HP1 | HP2 |
|---|---|---|---|---|---|
| `HEATER_NTU` | 2.649 | 2.539 | 2.512 | 2.479 | 2.445 |

정격 최종 급수온도 215.6℃ (실기 420℉). 부하가 낮으면 추기압이 낮아져
급수온도도 함께 내려간다.

**계열 격리** (NRC 2.6.2.6 / 2.6.2.8 — 저압·고압 모두 3계열 병렬):
병렬이라 정상 운전에서는 한 줄로 합쳐지지만, 한 계열을 잠그면 남은 계열이
같은 급수를 다 받는다. 계열당 유량이 늘고 전열면은 그대로라 NTU 가 줄어
TTD 가 벌어진다. 앞 단이 덜 데우면 뒤 단의 접근온도가 더 벌어지므로
**효과가 단마다 누적된다**.

| 계열 | LP1 | LP2 | LP3 | HP1 | HP2 | 최종 | 설계 대비 |
|---|---|---|---|---|---|---|---|
| 3 (설계) | 81.9 | 116.1 | 150.0 | 184.0 | 215.6 | 215.6℃ | — |
| 2 | 78.8 | 111.6 | 145.2 | 179.2 | 210.9 | 210.9℃ | −4.7℃ |
| 1 | 70.9 | 98.4 | 129.3 | 162.3 | 193.7 | 193.7℃ | −21.9℃ |
| 0 | 41.6 | 41.6 | 41.6 | 42.8 | 42.8 | 42.8℃ | 전 우회 |

급수가 차가워지면 노심 보이드가 줄어 **출력이 오른다** (감발이 아니다).
실기 해석사건 *loss of feedwater heating* 이다. 계열을 잠글 때 남은 계열의
압력손실이 유량 제곱으로 커지는 수력 제한은 모델에 없다.
`HEATER_PHI` = (0.78, 0.615, 0.469, 0.334, 0.218) — 각 추기점의 팽창 진행률
(7.03 MPa 팽창선에서 등엔트로피 역산).

> 원자로 쪽이 쓰는 급수온도는 이 계산 결과(`bop.feedwater_temp`)다. core 에는
> 정격 급수온도 상수가 없다 — 예전에 `FEEDWATER_TEMP` = 216.0 이 있었으나
> 참조되지 않는 데다 이 값과 0.4℃ 어긋나 지웠다.

### 10.4 복수기·순환수·냉각탑

```
cond_duty = (배기·바이패스·SJAE·드레인·RWCU가 복수기에 버리는 열) [MW]
cw_flow = CIRC_EACH(12230)·circ_running         143,400 gpm/대 = 9047, ×1.35 규모보정
cw_rise = cond_duty·1000/(cw_flow·CW_CP)          순환수 온도상승
cw_hot = cw_cold + cw_rise
eff = TOWER_EFF·(TOWER_NATURAL + (1-TOWER_NATURAL)·(fans/CELLS)^0.4)   팬 효과
cw_cold → cw_hot - eff·(cw_hot - wet_bulb)        냉각탑 출구 (1차지연)
cond_temp → cw_hot + COND_TTD·foul                복수기 포화온도 (비응축가스 악화)
cond_pressure = p_sat(cond_temp) + p_air          배압
```
- `TOWER_EFF` = 0.72, `TOWER_NATURAL` = 0.25(팬 정지 시 자연통풍 몫)
- 비응축가스(`p_air`)가 쌓이면 분압도 오르고 전열도 악화 → SJAE 멈추면 배압 상승
- **냉각탑은 의도적으로 매뉴얼과 다르다.** 매뉴얼의 발전소는 Long Island Sound
  해수를 한 번 쓰고 버리는 개회로 부지이고, 이 시뮬레이터는 내륙 부지를 가정해
  냉각탑 순환으로 바꿨다. `TOWER_*`·`WET_BULB`·`BASIN_*` 는 대조 대상이 아니다.

### 10.5 급수 가능유량·보호

펌프는 **직렬 세 단**이다 (NRC 2.6.2.2 / 2.6.2.5 / 2.6.2.7).

| 단 | 대수 | 용량/대 | 토출압 | 구동 |
|---|---|---|---|---|
| 복수펌프 | 2 × 50% | 1050 kg/s | `CONDENSATE_HEAD` 2800 kPa | 전동기 |
| 부스터펌프 | 2 × 50% | 1050 kg/s | `BOOSTER_HEAD` 4238 kPa (600 psig) | 전동기 |
| 급수펌프 | 2 × 50% | 1100 kg/s | 원자로압 + `FEEDPUMP_DP` | 증기터빈 |

```
cond_cap  = cond_running·CONDENSATE_EACH(1050) if 핫웰 수위 OK else 0
boost_cap = boost_running·BOOSTER_EACH(1050)   if cond_cap > 0 else 0

if boost_cap > 0: feed_head = BOOSTER_HEAD ; series_cap = min(cond_cap, boost_cap)
else:             feed_head = CONDENSATE_HEAD ; series_cap = cond_cap

rfp_steam_ok = main_steam > 0.5 and reactor_p > RFP_MIN_STEAM_P(1500 kPa)
if rfp_steam_ok:  feed_cap = feed_pumps·FEED_EACH(1100)
else:             feed_cap = series_cap·clip((feed_head - reactor_p)/500, 0, 1)
if rfp_tripped:  feed_cap = 0.0        # L8 고수위 래치 트립 (NRC 3.1.3.1.1)
feed_limit = min(series_cap, feed_cap)
```

**통과 유량은 가장 좁은 단이 정하고, 토출압은 마지막으로 승압한 단이 정한다.**
부스터를 다 세우면 흐르기는 하되(정지된 케이싱 통과) 승압이 없으므로 토출압이
`CONDENSATE_HEAD` 로 떨어진다.

급수펌프는 증기터빈 구동이라 주증기가 끊기면(MSIV 격리) 같이 멎는다. 그때는
앞의 두 단이 직접 원자로로 밀어 넣는데, `feed_head` 가 그 한계다. 이 값이
2800 이냐 4238 이냐가 격리 냉각 + 고압주입 상실에서 노심 노출을 가른다
(CHANGES §5.28). `feed_limit` 이 `_vessel_step` 의 급수 상한이 된다.

**터빈 보호/자동기동**:
- 저진공(`cond_pressure > COND_TRIP_PRESSURE` 27 kPa) → 터빈 정지
- 주증기 저압(`reactor_p < TURBINE_MIN_PRESSURE` 4 MPa) → 정지(자동복구)
- 자동기동: `reactor_p > TURBINE_ROLL_PRESSURE`(6.4 MPa) and 진공 OK → 동기

---

## 11. 검증 스크립트 대응표

| 스크립트 | 무엇을 확인 | 이 문서의 관련 절 |
|---|---|---|
| `tests/validate.py` | 정격 운전점·기동·스크램·붕괴열이 실기값과 맞는가 (`[7b]` 열적 제한) | §1, §2, §2.5 |
| `tests/subsys_test.py` | 계통별 개별 거동 (수위·비등·RWCU·MSIV·2차계통) | §2, §5, §10 |
| `tests/stress_test.py` | 사고 시나리오 물리범위 위반 감시 | 전체 |
| `tests/power_test.py` | 전원 상실 시 생존/정지, 버티는 시간 | §6 |
| `tests/diesel_test.py` | 디젤 자동기동·10초·용량제한·과부하트립·연료 | §6.3 |
| `tests/rbsw_test.py` | 서비스수 펌프·루프·격리·디젤냉각·유량한계 | §6.8 |
| `tests/atws_test.py` | ATWS — 붕산·재순환정지로 정지 가능한가, 멜트다운 | §2.2, §9 |
| `tests/slc_test.py` | SLC 설계제원·질량보존·정지시간 | §9 |
| `tests/shutdown_test.py` | ATWS+붕산+격리 1분 간격 계통 간 모순 | §2, §5, §9 |
| `tests/button_test.py` | 제어반 버튼이 실제로 계통을 바꾸는가 | 전체 (GUI) |
| `tests/gui_test.py` | 화면 없이 패널 자동 점검 | 전체 (GUI) |
| `tests/diagram_audit.py` | 계통도 배치 (겹침·끊김) | — |
| `tests/flow_test.py` | 계통도 흐름 표시가 실제 유량과 맞는가 | §2, §4, §7 |
| `tests/air_test.py` | 계장공기 순서·CRD 수압·제어봉 고착·RPS 전원·터빈건물 냉각수 | §3.0b, §3.1b, §3.3, §3.3b, §3.4 |
| `tests/nms_test.py` | SRM·IRM 눈금과 인터록, 24 VDC 계측전원 | §3.2, §3.2b |
| `tests/display_test.py` | 화면에 그려진 것과 계산된 것이 같은가 (잘림 포함) | 전체 (GUI) |
| `tests/conflict_test.py` | 이름 충돌·상수 중복 정의처럼 조용한 결함 | — |

---

## 12. 알려진 근사·한계 (물리 관점)

1. **연료 온도 1점**: 중심/표면 구분 없이 노드당 평균 하나. 노출 시 온도상승은
   §2.2에서 따로 계산.
2. **잉여반응도·제어봉 가치 역산값**: `EXCESS_REACTIVITY`(23281),
   `ROD_WORTH`(29500)는 실제 설계값이 아니라 정격 조건이 맞도록 역산.
3. **급수제어 2요소**: 증기유량+수위만 봄(실기는 급수유량도 보는 3요소).
4. **냉간 갇힌 공기**: `gas_mass`가 시뮬 내내 남음(실기는 기동 중 진공으로 배출).
5. **서비스수계통 단순화**: 원자로건물 서비스수(RBSW)를 펌프 4대·2루프·
   취수정 수온으로 모델링했다(§6.8). 다만 문헌의 폐회로 중간루프(RBCLCW)는
   두지 않아, 기기→RBSW 로 바로 연결한다. 터빈건물 TBSW/TBCLCW는 현재 구현되어 있다.
6. **Pm/Sm은 평균 모델**: I/Xe는 720점 공간 모델이며 Pm/Sm만 노심 평균으로 남는다.
7. **디젤발전기 단순화**: 3대 각 3500 kW 를 자동기동·10초·과부하트립·연료소모로
   모델링(§6.3). 부하는 안전 전동기 소비전력만 계상하고, 발전기 전압·주파수
   과도(로드 스텝 시 순간 강하)나 병렬운전 동기화는 다루지 않는다.
   직류전원(축전지)은 §6.4 에서 구현했다 — SBO 에서 축전지가 고갈되며
   RCIC 제어를 잃는 시점이 재현된다.
   외부전원 NSST/RSST 및 한 방향 고속절체는 현재 구현되어 있다.
   세부 전압·주파수 과도는 여전히 다루지 않는다.
8. **계장공기 모형**: 압축기·리시버·누설과 공기 상실에 따른 밸브 동작이
   구현되어 있다. 제어전원 상실과 별개인 공기 상실 경로를 검사한다.
9. **출력분포 지표는 운전 상태에 크게 의존한다**: `validate.py` 는 기동 직후
   (제어봉 90.4%)에 재므로 축방향 오프셋 -7.9% · 반경 최소비 0.59 가 나오지만,
   운전 평형(제논 평형·제어봉 98.4%)에서는 **-13.8% · 0.80** 으로 둘 다 목표
   안이다(§1.1 참고). 정격 지표를 인용할 때 어느 상태의 값인지 함께 밝혀야 한다.
   (`CORE_HEIGHT` 150 in · `TOP_OF_FUEL` -497.8 cm · `BOTTOM_OF_FUEL` -878.8 cm
   은 모두 매뉴얼 값과 일치한다.)
10. **열적 제한의 상관식이 대체품이다**: 임계출력은 GE 의 GEXL 로 계산해야
    하는데 그 식은 독점이라 공개되지 않는다. 그래서 **형태만 같은** 공개
    상관식(CISE-4, 임계건도 vs 비등길이)에 다발 보정계수 `CPR_K`(0.638)를
    곱해 정격 MCPR 1.45 가 나오도록 맞췄다(§2.5.2). 압력·유량 의존성의
    **방향과 기울기**는 상관식이 주므로 과도 거동은 신뢰할 만하지만,
    **절대값은 보정계수에 묶여 있다** — 다른 노심 설계로 바꾸면 다시 맞춰야 한다.
11. **다발 내부가 없다**: 채널 하나가 12.7 다발의 평균이고 다발 안의 봉별
    출력분포도 없다. 그래서 LHGR(봉 하나의 값)은 평면 평균(APLHGR)에
    `LOCAL_PEAKING`(1.13)을 곱해 추정한다. 실제 R-factor 나 모서리봉/내부봉
    구분은 표현되지 않는다. 또 **가장 뜨거운 실제 다발은 모델의 가장 뜨거운
    채널보다 더 뜨겁다**(12.7 다발이 평균되므로). 즉 MCPR 은 낙관적인 쪽이다.
12. **열적 제한의 연소·적용범위 보정**: 현재 LHGR·MAPLHGR은 누적 연소도를
    반영한다. MCPR 적용 범위 밖에서는 열출력 제한으로 전환한다. 상세 연료
    설계·다발별 제한곡선을 모두 모델링한 것은 아니다.
13. **`MAPLHGR_LIMIT`(11.2 kW/ft)의 근거가 약하다**: 매뉴얼은 값을 주지 않고
    실기 값은 기술지침서의 그래프로만 공개된다(DAEC Fig 3.12-8, 축 8~14 kW/ft).
    수치로 확인 가능한 가장 가까운 값이 Quad Cities COLR 의 11.00(GNF3, 10x10).
    다만 **축방향 반사체(§1.3)를 넣은 뒤로는 이 값이 결과를 가르지 않는다** —
    한계를 11.0 으로 잡든 11.5 로 잡든 MAPRAT 이 1.0 아래다(0.961 / 0.919).
    반사체 이전에는 이 한 값이 초과 여부를 결정했다.
14. **반사체가 축방향에만 있다**: 실기는 반경 방향에도 물 환상부가 있지만,
    모델은 반경 평탄화를 `FUEL_ZONE`(실측 분포에서 역산)이 맡고 있고 반경
    반사체는 +2021 pcm 이라 임계 조건을 다시 잡아야 해서 두지 않았다. 반경
    첨두 1.32 · 최소 0.81 은 규격(≤1.4 / ≥0.7) 안이다.


## 2026-09-13 제어·화면 해석 보충

열적 지표는 직접 RPS·인출차단 접점이 아니다. 인출은 APRM/RBM·SRM/IRM·RWM·
CRD·SDV 등을 검사한다. 수동 APRM/RBM 실험 우회는 자동과 RPS를 해제하지 않는다.
MSIV RUN 저압 설정은 825 psig에 대기압을 더한 5.7894998 MPa abs다. 기동 모드의
저압 우회와 다른 격리 조건을 구별한다. 증기관 상류 압력은 용기 압력으로 근사한다.

Recorder 125 눈금, LPRM 높이의 계산 Xe, 최고 피복재온도, OVF(+500 cm) 및 최고
수위 이력을 표시한다. 최고 연료 중심온도 하강만으로 MCPR 개선을 추론하지 않는다.
구체적인 재현 상태와 현재 검증 결과는 [보고서](tests/SPATIAL_FLOW_REPORT.md)를 따른다.

결합 복구는 두 부품의 위치 차이가 표시 반 눈금(0.05%p) 이내일 때만 허용한다. 떨어져 있는 날개가 복구
버튼으로 순간 이동하는 경로를 막는다. 최고 수위가 없는 구형 저장상태는 현재
수위부터 이력을 시작한다. 고배속에서 거절된 재순환 조작도 실제 속도가 아닌
원래 지령을 스테퍼에 다시 표시한다.

### 수조 수위가 정하는 두 가지 (R-104B Table 4.1-1 · 기술지침)

억제수조 수위는 표시용 숫자가 아니다. 두 군데에 직접 들어간다.

**① 다운커머 물기둥.** 드라이웰 가스가 수조로 밀려 들어가려면 다운커머
끝을 덮은 물기둥을 이겨야 한다.

```
물기둥 = ρ g h,   h = 잠김깊이
벤트 조건:  P_드라이웰 > P_압력억제실 + ρ g h
```

`h` 는 상수가 아니다. 수조 재고가 변하면 자유수면으로 나눈 만큼 따라간다.

```
h = h₀ + (V_수조 − V₀) / A_수면
h₀ = 1.2 m (약 4 ft),  A_수면 = 1003 m²
```

`A_수면` 은 Table 4.1-1 의 토러스 치수에서 나온다. 중심직경 111 ft,
단면직경 31 ft 이므로 중심선 둘레는 2π(55.5) = 348.7 ft 이고, 반쯤 찬
원형관의 수면 폭은 단면직경과 거의 같다(53% 충수에서 30.96 ft).
348.7 × 30.96 = 10,796 ft² = 1003 m². 검산: 토러스 전체 부피
2π²(55.5)(15.5²) = 263,200 ft³ 이 표의 물 135,000 + 기상부 119,000 =
254,000 ft³ 와 3.6% 안에서 맞는다.

마크I 기술지침이 토러스 수위를 좁은 띠(잠김 4.29~4.54 ft 같은)로 묶는
이유가 이것이다. 잠김이 얕으면 벤트 클리어링이 빨라져 하중이 커지고,
깊으면 클리어링 자체가 늦다. **SRV 를 4 대 한 시간 열면 잠김이 약
4 인치 움직인다 — 기술지침 띠 폭만 한 크기다.**

`h` 는 0 에서 자른다. 수조가 다운커머 끝 아래로 빠지면 벤트가 드러나
압력억제 기능 자체가 없어진다 — 음수 물기둥은 없다.

**② 압력억제실 기상부 체적.** 물이 불어난 만큼 기상부가 좁아진다.

```
V_기상 = V₀ − (V_수조 − V₀)
P_압력억제실 = P_질소(V_기상) + P_수소(V_기상) + P_포화(T_수조)
```

같은 질소가 더 좁은 데 갇히면 분압이 오른다. 개정판 Table 4.1-2 도
기상부를 '고수위에서' 와 '저수위에서' 두 값으로 적는다 — 고정이 아니다.

두 경로는 **같은 방향**으로 작동한다. 물이 차면 물기둥이 깊어져 벤트가
늦어지고, 기상부가 좁아져 압력억제실 압력이 올라 차압이 준다. 둘 다
벤트를 줄인다. (그래서 한쪽만 보는 시험은 다른 쪽이 죽어도 통과한다 —
CHANGES §5.70 에 그 일이 적혀 있다.)

### 격납용기 격리가 거는 연동 (R-304B 4.4)

격납용기를 둘러싼 계통은 사고 신호에 **자동으로 닫힌다.** 모델이 다루는
두 군(群)은 다음과 같다.

| 군 | 닫는 것 | 신호 |
|---|---|---|
| Group 8 | RBCLCW 격리밸브 · **드라이웰 유닛쿨러** | 고드라이웰압 · Level 1 · RBCLCW 헤드탱크 저저수위 |
| Group 9 | 격납용기 퍼지 · 질소봉입 | 고드라이웰압 · Level 2 · 재장전층 고방사선 · 원자로건물 저차압 |

**Group 8 은 되먹임이 고약하다.** 냉각기를 세우는 신호가 드라이웰이
더워졌다는 신호 그 자체라, 압력을 내릴 바로 그 설비를 압력이 세운다.
실기에서도 문제가 되어 NRC 정보통지 84-35 가 나왔다. 그래서 격리는
**래치**이고, 복귀는 신호가 없어야 받는다.

**퍼지 격리 우회는 교육용 조작이다.** 기존 Group 9 이름은 유지하되, 2차 격납
구현에서는 구판 R-104B의 직접 SGTS 흡입 경로를 선택했다. 고DW압/Level 2 격리
래치만 우회하며 실제 팬·계장공기·용량은 우회하지 않는다. 신판 §4.1.2.4.4의
사고 배기 서술만으로 구판의 실제 운전 절차나 우회 회로를 입증하지 않는다.

기존 **0.55 kg/s**는 신판 1000 scfm×질소밀도에서 가져온 가지관 상한으로
남긴다. 구판 SGTS의 정격으로 해석하지 않으며 이 조합은 NO BASIS다. 실제
유량은 혼합가스의 체적, SGTS 가용 팬 용량, 압력 하한으로 더 제한된다.

### 살수는 LPCI 와 같은 펌프를 쓴다 (R-304B 10.4.3.4)

RHR 은 하나의 계통이고, LPCI 주입·수조냉각·격납살수·정지냉각은 **같은
전동기**의 다른 정렬이다. 그래서 살수밸브는 LPCI 기동 신호에 물려
닫힌다. 노심이 아직 물을 필요로 하는데 그 물을 격납용기에 뿌리는 것을
막는 연동이다. 키락 우회로 열 수 있지만, 교범의 조건이 "노심냉각이
충분할 때" 다.

연동을 거는 것은 **신호**이지 펌프가 실제로 돌았느냐가 아니다. 자동기동
스위치를 꺼 두어도 신호가 뜨면 살수는 막힌다.

### 저부하에서 터빈은 증기를 휘젓는다 (R-304B 2.6.3.7 · Table 3.2-1)

정격에서 마지막 단 깃은 증기에서 일을 **받는다.** 부하가 낮아져 증기가
느려지면 그 관계가 뒤집혀, 깃이 증기에 일을 **준다** — 팬이 된다. 그
마찰·압축열은 빠져나갈 유량이 없어 배기후드에 쌓인다.

```
상승온도 = 풍손 / (배기유량 x 비열)
풍손 ∝ (회전수 / 정격회전수)^3          (팬 법칙)
```

유량이 크면 상승이 무시할 만하고(정격에서 약 2℃), 저부하에서 급격히
커진다. 터빈이 서면 회전수가 떨어져 풍손도 함께 사라진다.

교범이 사다리를 통째로 준다:

| | |
|---|---|
| 130 ℉ (54.4℃) | 살수밸브 개방 |
| 175 ℉ (79.4℃) | 운전원 경보 |
| 225 ℉ (107.2℃) | **주터빈 트립** (Table 3.2-1) |

**살수는 복수계통에서 온다** (2.6.3.8 — 후드 살수선은 복수 탈염기 하류
공통관에서 딴다. 같은 관에서 제어봉구동 펌프 흡입과 CST 배출도 나간다).
그래서 복수를 잃으면 살수도 없고, 같은 저부하가 터빈을 세운다.
**복수 상실이 후드를 거쳐 터빈에 닿는 경로**이고, 이것이 이 계통을 넣는
이유다.

⚠ 풍손 열량과 살수량은 교범에 없다. 위 사다리가 말이 되도록 역산한
값이라 상수 옆에 NO BASIS 로 적어 두었다. 설정치 셋은 근거가 있다.

### 진공을 잃을 때 무엇이 먼저 닫히는가 (3.2.4 · 7.3.3.2.8)

복수기 진공이 무너지면 두 가지가 닫힌다 — 터빈 정지밸브와 바이패스밸브.
**순서가 설계의 핵심이다.**

```
압력이 오르는 방향 →
  20.0 kPa   저진공 경보
  25.1 kPa   터빈 트립       (Table 3.2-1, 22.5″Hg vacuum)
  30.0 kPa   MSIV 격리       (이 모델의 DELIBERATE 신호)
  77.6 kPa   바이패스 닫힘    (3.2.4, 7″Hg vacuum)
```

터빈이 먼저 서고, 바이패스는 그 뒤로도 한참 열려 있다가 복수기가 거의
대기압이 되어서야 닫힌다. 과도 중에 증기를 버릴 곳을 최대한 오래 남기려는
것이다. 옛 판이 이유를 적는다 — **복수기를 과압에서 보호**하기 위해서다.
진공이 없는 복수기에 증기를 계속 버리면 쉘이 대기압을 넘는다.

⚠ 단위를 조심한다. 7″Hg 를 **절대압**으로 읽으면 23.7 kPa 이 되어 터빈
트립보다 낮아지고, 사다리가 통째로 뒤집힌다. 개정판 강의본이 못박는다 —
"less than 7 inches mercury **vacuum**".

## 2026-09-20 RBCLCW — 원자로건물 폐회로냉각수

`core/building_systems.py`의 `RBCLCW`가 폐회로 물·펌프·열교환기·부하 열상태를
소유한다. `Reactor`가 전원, RBSW, Group 8, DW, RWCU 및 압력용기와 연결한다.
`Reactor.rbclcw_status(load)`는 물리·제어반·계통도가 같이 쓰는 부하별 공급
판정이다. DW에는 Group 8 관통부 격리까지 포함한다. 2차 격납·SGTS는 별도 작업이다.

### 근거와 채택값

구판 R-104B §11.3과 신판 R-304B §11.3을 대조했다. 세부 동작은 신판
§11.3.5.1~4, 부하 구분은 Table 11.3-1을 따른다. 자료와 웹 대조의 적용 범위는
`PLAN_RBCLCW_SECONDARY.md` §6에 있다. 교범 설비를 3293/2436으로 일괄 확대하지 않는다.

| 항목 | 채택값 | 근거 또는 모형 가정 |
|---|---|---|
| 주펌프 | 3대 × 50%, 각 1600 gpm = 100.94432 kg/s | 양 교범 §11.3; 정상 A/B 운전·C 대기 |
| 주 HX | 2대 × 100%, 정상 1대 | R-304B §11.3.4.3 |
| 흡입수 목표 | 91°F = 32.7778°C | R-304B §11.3.5.1; 우회 제어 목표 |
| C 재기동 제한 | 복전 후 600초 + 운전원 재선택 | R-304B §11.3.5.2 |
| M/G 냉각수 펌프 | 2대, 저흡입 10 psig에서 트립 | R-304B §11.3; 주 재순환펌프와 다른 기기 |
| RWCU NRHX 출구 | 140°F = 60°C, MOV-034 격리 | R-304B §2.8.4.4 |
| RWCU 펌프 냉각수 출구 | 195°F = 90.56°C, 별도 펌프 트립 | R-304B §2.8.4.5 |
| 전동기 입력 | 주펌프 82.856 kW/대, M/G 냉각펌프 62.142 kW/대 | 구판 100/75 hp × 0.7457 ÷ 효율 0.90; 효율은 NO BASIS |
| 480 V 급전 | A/B/C를 비상모선 101/102/103에 대응 | NO BASIS: 기존 추상 모선에 배정; 실제 배선도라는 뜻이 아님 |
| 물 재고 | 배관 10000 kg + 헤드탱크 최대 500 kg/루프 | NO BASIS; 정상 탱크 50%, 저저 10% |
| 유량 저하 | 탱크 5% 이하에서 감소, 0%에서 순환 불가 | NO BASIS: NPSH 상세 계산의 대용; 별도 저수위 자동트립 아님 |
| HX | UA 250 kW/K/대, SW 배정 최대 100 kg/s/대, 승온 8 K | NO BASIS; 제열량 약 3.34 MW/대의 서비스수 상한 |
| 온도 제어 | 목표 복귀 시정수 30초, 보충 최대 2 kg/s·20°C | NO BASIS; 공기 상실 때 우회 닫힘·보충 열림 |

### 사고 신호·전원·복귀

고DW압 또는 Level 1은 안전 A/B를 분리하고 비안전 루프를 격리하며 HX를
정렬한다. 분리 뒤 A는 B의 재고·서비스수를 빌리지 못한다. C는 공통 예비로
간주하므로 분리 뒤 안전 루프를 대신 급수하지 않는 모형 가정을 적용한다.
헤드탱크 저저수위는 **Group 8의 관통부·DW 냉각기 격리에만** 추가하고 전체
LOCA 분리 신호에는 넣지 않는다(R-304B §4.4와 §11.3의 서로 다른 동작 범위).

무전원 MOV는 실제 위치를 유지한다. 급전 없는 HX 조작·격리 복귀는 거절한다.
A/B는 자동 선택일 때 복전 기동하고, C는 600초 뒤에도 재선택 전까지 정지한다.
LOCA 정렬은 전원이 있는 밸브만 움직인다. 밸브 이동 시간과 개별 직렬 MOV는
등가 경로로 묶었다. 유지 신호 중 복귀 거절, 복귀 버튼의 묶음 범위는 교육용
조작 정책이다. 팬 복귀만으로 물측 격리가 열리지 않는다.

공기 상실 시 HX 출구 열림·우회 닫힘·탱크 LCV 열림 방향을 반영한다. 보충은
탱크 최대 재고까지만 계산하며 넘침 배관은 생략했다. PCV-71 닫힘의 압력곡선,
보조(booster) HX의 별도 물 재고·밸브군은 상세 모델이 없다. DW·CRD는 비안전
냉각 경로에 열적으로 합산된다. RBSW의 신판 전체 밸브 정렬을 구현했다고
주장하지 않으며 기존 RHR 모드 선택 정책을 유지한다.

### 열·물·전력 수지

각 루프는 `E = m·4180·T`를 저장하고 `dE = (Q부하−QHX)dt + 유입−유출
엔탈피`로 갱신한다. 누설은 실제 물 재고를 한도로 빼고 그 온도의 엔탈피도
뺀다. 보충수는 20°C 엔탈피로 들어간다. 정상 크로스커넥트는 질량·엔탈피를
보존하며 혼합하고 분리 후에는 혼합하지 않는다. 누계 장부로 수지를 시험한다.

HX 제열은 `UA·max(T−TSW,0)`와 `배정 SW유량·cp·8 K` 중 작은 값이 상한이다.
우회가 그 범위 안에서 91°F를 제어한다. 공급 서비스수가 없으면 HX 제열은
0이지만 폐회로 순환·물의 열용량은 남는다. 뜨거운 취수로 목표보다 차갑게
만들 수 없고, 한 스텝에서 취수온도 아래로 지나치게 냉각하지 않는다.

DW의 **실제** 제거열을 같은 양으로 폐회로에 더한다. 공급수 가열·무유량·
Group 8 격리에 따라 DW 제열이 감소한다. 재순환 전동기/축봉은 루프별 열용량
1 MJ/K, 전도도 최대 8 kW/K, 회전손실 75 kW×속도², 고온수 축봉열 최대 약
5 kW/대의 집중 모형이다. M/G는 1 MJ/K·12 kW/K·60 kW×두 속도 합, CRD는
실제 급전 펌프당 20 kW, RHR 축봉은 선택된 운전대수당 5 kW로 근사한다.
모두 **NO BASIS**이며 교범 설계 열부하가 아니다. RHR 축봉 개별 온도·고장은
계산하지 않는다. 재순환 주펌프의 임의 지연 트립을 만들지 않는다.

RWCU NRHX는 0.3 MJ/K, 열측 4 kW/K, 냉각측 최대 50 kW/K의 집중 모형이다.
열교환 해석해의 평균 온도로 물측 전달열을 계산하고 같은 열측 에너지를
RPV에서 뺀다. 펌프 냉각수 출구는 공급수+운전 시 4 K를 30초로 추종하는
검출점 근사다. **NRHX 격리와 펌프 트립은 별도 래치**다. 시간 지연은 온도
계산 결과이며 실제 설비의 고장 후 허용시간으로 해석하지 않는다.

서비스수는 DG 운전당 20 kg/s(NO BASIS)를 먼저 배정하고 RBCLCW HX가 배정받은
몫을 뺀 나머지를 RHR에 제공한다. 수조냉각·살수·정지냉각도 이 남은 열수송
한도를 공유한다. 현재 계산 순서상 수조냉각/살수가 먼저 쓰고 SDC가 나머지를
쓴다. 부하별 실제 조절밸브 압력망을 대체하는 보수적 배분 가정이다. 전력은
실제 운전 중인 주펌프·M/G 냉각펌프만 자기 모선 부하에 더한다.

계수 민감도 확인은 UA·루프 재고·RWCU 냉각측 전도도를 각각 0.5~2배로 바꿔
정상 유지와 상실 후 온도 상승 시간을 비교한다. 온도 상승 **방향**, 유량과
열침원의 구분, 질량/엔탈피 보존, 신호별 동작 범위가 검증 대상이며 특정 초의
고장시간을 실기 기준으로 맞춘 모형은 아니다. 실행 결과와 미실행 항목은
`tests/RBCLCW_REPORT.md`에 기록한다.

## 2026-09-20 2차 격납 — 필요한 요소만 구현한 SGTS

`core/secondary_containment.py`는 **R-104B 4.2·4.3의 SGTS 3계열**을 선택한
단일 건물 모형이다. 신판 R-304B §4.2·4.3의 RBSVS 2계열, 정상 -1.5 inH2O,
저차압 -0.30 inH2O/30초 기동, booster fan과 혼합 환기를 섞지 않는다.
[NRC Issue 192](https://www.nrc.gov/sr0933/section-3-new-generic-issues/issue-192-secondary-containment-drawdown-time)
도 2/3계열 배열이 부지별로 다르며 한 공간의 음압이 모든 방의 음압을 보장하지
않음을 설명한다. 세 계열을 한 호기의 한 건물에 대응시킨 것은 교육용 축약이다.

| 요소 | 채택값·근거 또는 가정 |
|---|---|
| 정상 차압·음압 판정 | -0.25 inH2O = -62.275 Pa; R-104B §4.2.1, §4.3.4.2 |
| SGTS 팬 | 3계열, 팬당 약 9000 scfm; R-104B §4.3.1, §4.3.2.8 |
| 자동 신호 | 저수위·고DW압·연료교체층 고방사선·건물 환기배기 고방사선; §4.3.3.2 |
| 저수위·고DW압 수치 | 기존 Level 2 / DW_HIGH_PRESSURE 사용; 구판 §4.3은 수치 미제시 |
| 정상 환기 격리 | 위 신호에 급·배기 팬 정지, 댐퍼 닫힘; §4.2.3.2·§4.3.3.2 |
| 퍼지 경로 | DW → SGTS 직접 흡입 → 굴뚝; §4.3.2.1·§4.3.4.1 |
| 건물 체적·온도 | 100000 m³, 20℃ 고정; **NO BASIS** |
| 외벽·문 전도도 | 0.02 / 문 경계 상실 시 추가 0.8 standard m³/s/Pa; **NO BASIS** |
| 정상 급·배기 | 급기 20, 배기 21.2455 standard m³/s; 기본 누설과 -0.25에서 평형, **NO BASIS** |
| 팬 곡선 | 건물 흡입부 차압 0~-1 inH2O에서 용량이 선형 감소; **NO BASIS** |
| 팬 전원·부하 | A/B/C → 비상모선 101/102/103, 운전 팬당 30 kW; **NO BASIS** |
| 밸브 구동 | 정상 댐퍼는 정상 AC/공기 상실 시 닫힘, SGTS는 해당 AC, 퍼지는 공기 필요; **NO BASIS** |
| 복귀 | 신호 해소·수동 팬 자동/정지 전환·정상 AC/공기 복구 후 수동 복귀; 교육용 묶음 |

표준 유량의 상태는 이 모형에서 20℃·101325 Pa로 통일한다. `P_g = P_atm ×
(m_air/(rho_air×V) - 1)`이며 질량 변화는 급기 + 틈새 유입 - 정상 배기 - SGTS
건물 배기 - 틈새 유출이다. 팬은 압력을 직접 지정하지 않는다. 팬 곡선의
선형 구간마다 틈새 교환과 배기를 함께 해석적으로 적분하고 구간 경계를
넘을 때만 나눈다. 매 0.25초의 불필요한 내부 반복은 제거했다.
`initial_mass + air_in - air_out - air_mass`가 공기 수지 잔차다.
전 팬 정지 시 틈새로 대기압에 접근하고, 양압에서는 유출 부호로 바뀐다.
문 고장은 이중문 양쪽이 동시에 열린 경계 상실이며 정상 출입을 뜻하지 않는다.

SGTS 기동 요구와 격리 래치는 운전 가능 여부와 별개다. 정지·고장·무급전은
실제 팬만 막는다. 전원 복구 뒤 자동 계열은 남아 있는 요구로 재기동한다.
수동 운전은 그 팬만 켜되 정상 환기를 격리한다. 방사선 두 입력은 독립 시험
접점이고 `clad_damaged`나 수소량에 직결하지 않는다. **저음압은 표시 경보이며
SGTS/RPS 자동 기동 신호가 아니다.** R-304B Group 9의 네 입력을 모두 구현했다고
주장하지 않는다. 기존 `group9_signal()`은 구판 공통 두 공정 신호를 유지한다.

퍼지 가지관의 기존 0.55 kg/s 상한은 보존하되 SGTS 가용 용량을 넘지 못한다.
혼합가스 표준 체적은 `m_H2/rho_H2 + m_N2/rho_N2`다. 이를 팬 정격에서 먼저
빼고 남은 용량·흡입 압력으로 건물 배기를 계산한다. 이 배분 순서는 **NO BASIS**
단순화이며 덕트 압력망은 아니다. DW 가스는 건물 공기에 더하지 않고 곧바로
SGTS 흡입관을 지나므로 `h2_purged`는 굴뚝으로 실제 배출된 누적 수소다.
기존 `h2_balance()` 경계는 그대로이며 외부 배출을 두 번 세지 않는다.
필터가 N2/H2를 소멸시키거나 가스 재고를 생성하는 항은 없다.

최소 범위에서 **생략**한 것: 구역별 압력·열, DW 누설/파손의 건물 유입,
HPCI 글랜드 배기 유량, HEPA/활성탄 제거효율·습도·방사능·피폭량·연소·화재·
offgas, 실제 정상 퍼지 전용 팬/덕트와 독립 integrity-test 모드. HPCI 글랜드가
SGTS에 연결된다는 문헌 사실(§4.3.4.3)만으로 임의 증기 유량을 만들지 않는다.
특히 이 단일 공간 가정의 음압 형성시간을 실기 합격 기준으로 사용하지 않는다.

검증은 질량 보존, 신호별 실제 동작, 전원·고장, 문 상실, 정상 유지와 시간 간격
수렴을 대상으로 한다. 실행 결과는 `VERIFICATION.md` §13.37에 기록한다.
