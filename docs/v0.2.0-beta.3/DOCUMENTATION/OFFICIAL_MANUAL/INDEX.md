# NRC BWR/4 Technology Manual (R-104B) — 검색 인덱스

추출: `pdftotext -layout` → `part1.txt` ~ `part4.txt` (총 ~442쪽, OCR 스캔본)

> PDF 원본 4개(약 20 MB)는 용량 때문에 저장소에 넣지 않았다.
> NRC 공개 문서라 누구나 받을 수 있다. 여기에는 grep 할 텍스트만 둔다.

> ### ★ 먼저 개정판을 본다 — [`R304B_INDEX.md`](R304B_INDEX.md)
>
> 같은 교범의 후속 개정판(R-304B, Rev 09/11) **전문이 `r304b/` 에 들어와**
> 있다. 이 파일(R-104B)은 우리가 OCR 한 스캔본이라 2 단 조판이 섞이고
> 소수점이 떨어지며, **표가 상당수 그림이라 텍스트에 아예 없다.**
> 개정판은 NRC 가 뽑은 텍스트라 그 셋이 다 낫다.
>
> 그래서 **"이 매뉴얼에 근거가 없다" 는 판정은 개정판을 확인하기 전까지
> 미완이다.** 실제로 Table 9.4-1~3(계열별 직류 부하)이 그렇게 빠져 있었고,
> 개정판에서 찾아 §5.65 를 고쳤다. 절 번호로 바로 여는 법:
>
> ```bash
> grep -P "^9\.4\.3" manual/R304B_SECTIONS.tsv
> ```

> **전체 검증 결과는 [`../VERIFICATION.md`](../VERIFICATION.md) 에 있다.**
> 이 파일은 **매뉴얼을 찾기 위한 색인**이고, 그쪽이 **대조 결과와 판정**이다.
> 값이 서로 다르면 `VERIFICATION.md` 가 최신이다.

## 사용법

```bash
cd manual                            # 저장소 안
grep -n "찾을내용" part*.txt          # 전체 검색
sed -n '1234,1280p' part3.txt        # 해당 위치 읽기
```

**OCR 주의**
- 2단 조판이라 좌/우 단이 한 줄에 섞여 나온다. 문맥이 끊기면 앞뒤 20줄을 더 본다.
- 페이지 머리글은 심하게 깨진다 (`General Electric Technolov Manual`). 본문은 대체로 정확.
- **소수점이 자주 탈락한다**: `10.1.3.2` → `10.13.2`, `2.5.2.3` → `2.5.2.`.
  섹션 번호로 grep할 때는 `10\.1\.3` 처럼 느슨하게 잡고, 제목 단어로도 함께 찾을 것.
- 숫자 OCR 오류 가능(`l0` = 10, `-70 psig` = ~70 psig). 중요한 수치는 문맥으로 교차확인.
- **부호가 자주 사라진다**: 반응도계수 `α_V -1 X 10-3` 이 `axV I X 10-3` 으로 나온다.
  본문 서술("aV is negative")로 부호를 확인할 것.

## 곁다리 자료 — 매뉴얼에 없는 것을 찾을 때

이 매뉴얼(R-104B)로 부족할 때 실제로 답이 나온 문서들. 전부 NRC 공개자료다.

| 문서 | 무엇이 있나 | 링크 |
|---|---|---|
| **R-304B 1.8 Thermal Limits** — **같은 매뉴얼의 후속 개정판** | 열적 제한이 훨씬 상세하다. GEXL 방법론(접선법), 운전한계 MCPR 1.44, Table 1.8-1 BWR 노심 제원 비교(BWR/4 = **BFNPP 3293 MWt**), Table 1.8-3 전산기 감시비율(MFLCPR·MFLPD·MAPRAT) | [ML11258A297](https://www.nrc.gov/docs/ML1125/ML11258A297.pdf) |
| **Duane Arnold(DAEC) 기술지침서 3.12** — **BWR/4 · Mark I 실기** | LHGR 13.4 kW/ft(8x8) 확인, MAPLHGR 저유량 95% 규칙, MAPLHGR 곡선(Fig 3.12-8, 그래프라 값은 못 읽음) | [ML112230830](https://www.nrc.gov/docs/ML1122/ML112230830.pdf) |
| **Quad Cities 1 COLR** | MAPLHGR 실수치표(GNF3: 0~38.64 GWd/ST 에서 **11.00 kW/ft** 평탄), OLMCPR·Kp·MCPRf 표 | [ML23101A065](https://www.nrc.gov/docs/ML2310/ML23101A065.pdf) |

> **개정판이 이 매뉴얼과 다른 곳이 있다.** LHGR 1% 변형 한계가 20 kW/ft 로 내려가는
> 노출을 R-104B 는 "40,000 **MWd/MT**", R-304B 는 "40,000 **MWd/sT**" 로 적는다
> (약 10% 차이). 시뮬레이터는 연소도 모델이 MT 기준이라 R-104B 를 따랐다.

> **표 안에서도 연료형식이 섞인다.** R-304B Table 1.8-1 의 BWR/4 열 평균 LHGR
> 7.05 kW/ft 는 **7x7**(49봉) 값이다. 같은 표의 BWR/5·BWR/6 은 8x8 이라 첨두가
> 13.4 로 나온다. 열 제목이 발전소 이름이지 연료형식이 아니다.

PDF 는 `pdftotext -layout` 로 뽑아야 읽을 수 있다 (WebFetch 는 원시 PDF 를 못 읽는다).

## 파트별 장 구성

| 파트 | 장 | 내용 | 줄 수 |
|---|---|---|---|
| **part1** | 1.x, 2.x | 서론·원자로물리·열적제한 / 1차·보조계통 | 5,822 |
| **part2** | 3.x, 4.x, 5.x, 6.x | 공정계측제어 / 격납용기 / 중성자계측 / 전산 | 4,606 |
| **part3** | 7.x, 8.x, 9.x, 10.x | 반응도제어 / 방사성폐기물 / 전기 / ECCS | 4,536 |
| **part4** | 11.x, 12.x, 13.x | 용수·공기 / 운전 / BWR 차이점 | 2,967 |

**전체 목차는 `part1.txt` L380–528** 에 통째로 있다.

---

## ⚠ 매뉴얼은 한 발전소가 아니다 — 값을 쓰기 전에 반드시 읽을 것

서문:

> *"The data provided are **not necessarily specific to any particular nuclear
> power plant**, but can be considered to be **representative of the vendor design**."*

**장마다 규모가 다르다.**

| 출처 | 규모 |
|---|---|
| 9.1 소내전원 | *"The unit generator supplies **880 megawatts**"* |
| Figure 2.0-2 열수지 | 증기 10.5×10⁶ lb/hr ≈ **2436 MWt** 급 |
| 11장 (순환수·서비스수) | Long Island Sound 해수 부지 |
| **Table 1.5-1 발전소 목록** | **3293 MWt · 1065 MWe · 764 다발 · 185 제어봉** ← **시뮬레이터 기준** |

**그래서 값을 두 종류로 나눠 쓴다.**

| 종류 | 규모 의존 | 사용법 |
|---|---|---|
| 시간 (ADS 105 s, 디젤 10 s) | ✗ | **그대로** |
| 압력 (150/1150 psig, 0.5 psi) | ✗ | **그대로** |
| 설정치 기울기 (0.66W+42% 차단 / +51% 스크램), 대수, 비율 (30%) | ✗ | **그대로** |
| **유량 · 전력 · 부피** | **✓** | **× 3293/2436 = 1.35** |

> **매뉴얼 수치를 코드에 넣기 전에 "이 값이 발전소 크기에 비례하는가" 를 먼저 묻는다.**
> 비례하면 어느 장에서 왔는지 확인하고 1.35 배 환산한다.
>
> 결정적 단서였던 것: **LPCI 만 매뉴얼과 코드가 10,000 gpm 으로 같았다.** RHR 펌프는
> 규모와 무관하게 표준화돼 있기 때문이다. HPCI·RCIC·CS 가 전부 어긋난 것은
> 환산 실수가 아니라 발전소 크기 차이였다.

---

## 시뮬레이터가 쓰는 절 — 빠른 위치표

### 1장 원자로 물리·열적 제한

| 주제 | 절 | 파일 · 줄 |
|---|---|---|
| **원자로 물리 (장 시작)** | 1.12 | part1 **L1437** |
| ├ 감속재온도계수 α_T | 1.12.5.1 | part1 L1499 |
| ├ 보이드계수 α_V | 1.12.5.2 | part1 L1513 |
| ├ 도플러계수 α_D | 1.12.5.3 | part1 L1523 |
| ├ **반응도계수 수치 3종** ("Approximate numerical values") | 1.12.5.4 | part1 **L1495** (우단) |
| ├ **제논 (9.2 h / 6.7 h 반감기)** | 1.12.7.1 | part1 **L1639** |
| └ **사마륨-149** ("Next to xenon-135, the most important…") | 1.12.7.2 | part1 **L1666** |
| **열적 제한 (장 시작)** | 1.13 | part1 **L1741** |
| ├ MAPLHGR 개요 | 1.13.x | part1 L1783 |
| ├ **LHGR 한계** (제목 L1918 좌단) | 1.13.4 | part1 **L1918** |
| │  └ 값: 25 → **20 kW/ft @ 40,000 MWd/MT** · **설계 13.4 kW/ft** | 1.13.4 | part1 **L1902–1906** (우단) |
| ├ APLHGR 한계 | 1.13.5 | part1 **L1911** (우단) |
| └ **MCPR 안전한계 1.07** | 1.13.6 | part1 L1964 (제목) / **L1980** (값) |
| **Table 1.5-1 발전소 목록** (3293 MWt 행) | 1.5 | part1 (규모 기준) |

> **1.13 은 2단 조판이 심하다.** 좌단(제목)과 우단(값)의 줄번호가 어긋난다 —
> 1.13.4 제목은 L1918 인데 그 본문 값은 L1902–1906 우단에 있다. 좌단을 다 읽은 뒤
> 우단을 읽는 순서다.

### 2장 1차·보조계통

| 주제 | 절 | 파일 · 줄 |
|---|---|---|
| 압력용기 | 2.1 | part1 L2270 |
| **연료·제어봉** | 2.2 | part1 L3140 |
| └ **"active fuel length of 150 inches"** | 2.2.2.1 | part1 **L3304** |
| 제어봉구동 (CRD) | 2.3 | part1 L3628 |
| 재순환계통 | 2.4 | part1 L4145 |
| **주증기계통 (MSIV·SRV·터빈밸브)** | 2.5 | part1 L4422 |
| ├ SRV | 2.5.2.1 | part1 L4428 |
| ├ 유량제한기 | 2.5.2.2 | part1 L4572 |
| ├ **MSIV** (air to open / air+spring to close) | 2.5.2.3 | part1 **L4586** |
| ├ **바이패스 4대 30%** | 2.5.2.5 | part1 **L4607** |
| ├ 터빈 정지밸브·제어밸브 | 2.5.2.6–7 | part1 L4577 |
| ├ 터빈 (1800 RPM tandem compound) | 2.5.2.8 | part1 L4599 |
| ├ **정지/제어밸브 폐쇄 → 스크램 + RPT** | 2.5.3.2 | part1 **L4701** |
| ├ HPCI/RCIC 증기원 (RCIC=A, HPCI=B) | 2.5.4.3–4 | part1 L4760~ |
| └ **SRV 대수 "six of the thirteen"** | 2.5.4.11 | part1 **L4766** |
| 복수·급수 | 2.6 | part1 L4879 |
| ├ 복수펌프 2대 50% | 2.6.2.2 | part1 L5010~ |
| ├ **복수 여과·탈염기** | 2.6.2.4 | part1 **L5039** (좌단) |
| ├ **복수 부스터펌프 2대 (600 psig)** | 2.6.2.5 | part1 **L5030** (우단) |
| ├ 급수가열기 (DC + LP×3) | 2.6.2.6 | part1 **L5039** (우단) |
| ├ 급수펌프 (가변속 증기터빈 구동) | 2.6.2.7 | part1 **L5063** |
| └ 급수펌프 기동압력 "approximately 300 psig" | 2.6.3.1 | part1 **L5068** (우단) |
| **RCIC** (증기압 하한 150 psig) | 2.7 | part1 L5238 |
| RWCU | 2.8 | part1 L5492 |

### 3장 공정계측제어

| 주제 | 절 | 파일 · 줄 |
|---|---|---|
| **수위계측 + 수위 설정치 L1~L8** | 3.1 | part2 L139 / **설정치 L260·L295** |
| ├ L8 트립 3종 (주터빈·급수펌프터빈·HPCI/RCIC) | 3.1.3.1.1 | part2 L260~ |
| ├ **L4 + 급수펌프 트립 → 재순환 런백** | 3.1.3.1.4 | part2 L275~ |
| ├ L2 근거 + ATWS-RPT | 3.1.3.1.6 | part2 L290~ |
| └ L1 근거 ("TAF 보다 충분히 위") | 3.1.3.1.7 | part2 L295~ |
| **EHC (압력조절)** | 3.2 | part2 L511 |
| ├ 압력조절기 (920 psi 설정, 30 psi/100%) | 3.2.2.1 | part2 **L569** |
| ├ 부하제어 (load limit 100%) | 3.2.2.2 | part2 L625 |
| └ **밸브제어 — 바이패스 = 압력지령 − LVG출력** | 3.2.2.4 | part2 **L615** |
| 급수제어 (3요소) | 3.3 | part2 L784 |

### 4~6장 격납용기·중성자계측

| 주제 | 절 | 파일 · 줄 |
|---|---|---|
| **1차격납용기 (Mark I)** | 4.1 | part2 L1087 |
| ├ **Table 4.1-1 Mark I 제원표** | 4.1 | part2 **L1460–1495** |
| │  ├ 드라이웰: 자유체적 159,000 ft³ · 설계압 62 psig · 설계온도 281°F | | part2 **L1468–1474** |
| │  └ 압력억제실: **수량 135,000 ft³** · 기상부 119,000 ft³ · 62 psig · 281°F | | part2 **L1481–1489** |
| ├ 진공파괴밸브 0.5 psi | 4.1.2.3 | part2 L1105~ |
| ├ 질소 봉입·퍼지 | 4.1.3.3 | part2 L1285 |
| ├ 드라이웰/압력억제실 차압제어 | 4.1.3.5 | part2 L1113 |
| └ LOCA 시 격납 거동 | 4.1.3.8 | part2 L1119 |
| 2차격납 / SGTS / 격납격리 | 4.2–4.4 | part2 L1551·L1833·L2068 |
| 중성자계측 (SRM·IRM·LPRM·APRM·RBM·TIP) | 5.1–5.6 | part2 L2532~L3579 |
| RWM / RSCS / 공정전산 | 6.1–6.3 | part2 L3851·L4178·L4375 |

### 7장 반응도제어

| 주제 | 절 | 파일 · 줄 |
|---|---|---|
| **제어봉 수동제어 + Table 7.1-1 인출차단** | 7.1 | part3 L85 / **표 L268** |
| **재순환 유량제어** | 7.2 | part3 L508 |
| └ **EOC-RPT / ATWS-RPT** (30%, 1120 psig, L2) | 7.2.3.2 | part3 **L677** |
| **원자로보호계통 (RPS) — 스크램 설정치** | 7.3 | part3 L863 |
| └ **Table 7.3-1 스크램 신호표** | 7.3 | part3 **L1188** |
| **SLC (붕산주입)** | 7.4 | part3 L1319 |
| ├ 탱크 4,850 gal | 7.4.2.1 | part3 **L1413** |
| ├ 펌프 2대 100% / 39 gpm | 7.4.2.2 | part3 **L1364** (우단) |
| └ 기동 동작 + "not a backup for scram" | 7.4.3.1 | part3 **L1437** |

### 8~9장 방사성폐기물·전기

| 주제 | 절 | 파일 · 줄 |
|---|---|---|
| 오프가스 / 방사성폐기물 / 방사선감시 | 8.1–8.4 | part3 L1571~L2158 |
| **정상 소내전원** ("880 MWe" ← 규모 단서) | 9.1 | part3 **L2419** |
| └ 발전기차단기·주변압기·switchyard | 9.1.2.1–3 | part3 L2450~ |
| **비상 교류전원 (모선 3계열 + 디젤 3대)** | 9.2 | part3 **L2636** |
| ├ 모선 101 Red / 102 Blue / 103 Orange, 연결 없음 | 9.2.2.1 | part3 L2660~ |
| ├ 디젤 연속정격 3500 kW · 기동 <10초 | 9.2.2.2 | part3 L2690~ |
| ├ 디젤 자동기동 4신호 · **Table 9.2-1 부하표** | 9.2.3.2 | part3 L2716 |
| └ **외부전원 고속절체 (NSST→RSST, ~5 사이클)** | 9.2.3.3 | part3 **L2735** ※ |
| **120 VAC** | 9.3 | part3 L2843 |
| └ **UPS 구성 + "minimum of two hours"** | 9.3.1.4 | part3 **L2861** |
| **직류전원 (125V 축전지, division당 2대+2충전기)** | 9.4 | part3 L3033 |

> ※ **9.2.3.3 은 목차와 본문 제목이 다르다.** 목차(part3 L2616)는 *"Shutdown Board
> Loading"*, 본문(L2735)은 *"Loss of Preferred Power (LOPP)"* 다. 외부전원 절체를
> 찾을 때는 본문 쪽을 볼 것.

### 10장 ECCS

| 주제 | 절 | 파일 · 줄 |
|---|---|---|
| ECCS 개요 (HPCI 정지밸브 ~100 psig) | 10.0.7 | part3 L3400~ |
| **HPCI** | 10.1 | part3 L3439 |
| ├ 운전범위 상한 1150 psig | 10.1.2 | part3 L3480~ |
| └ **자동 터빈트립 5조건** (⑤ 고수위) | 10.1.3.4 | part3 **L3625** |
| **ADS** | 10.2 | part3 L3760 |
| └ **105초 타이머** (본문 5회 + Figure 10.2-1) · SRV "Seven of the eleven" | 10.2.3.1 | part3 L3800~ |
| **노심살수 (CS)** — 정격토출압 274 psig | 10.3 | part3 L3964 |
| **잔열제거 (RHR)** | 10.4 | part3 L4185 |
| └ **RHR 펌프 10,000 gpm · 정격토출압 136 psig** · 모선 배분 | 10.4.2.2 | part3 **L4254** |

### 11~13장 용수·운전

| 주제 | 절 | 파일 · 줄 |
|---|---|---|
| **순환수계통** | 11.1 | part4 L88 |
| └ **펌프 143,400 gpm · 전동기 1,500 hp** | 11.1.2.1 | part4 **L99** (값 L103–105) |
| **원자로건물 서비스수 (RBSW)** | 11.2 | part4 **L229** |
| ├ **펌프 4대 8600 gpm · 450 Hp** | 11.2.2 | part4 **L267** (값 **L234–235**) |
| ├ 2루프 (A또는C / B또는D) | 11.2.3.1 | part4 **L244** |
| └ 디젤 엔진냉각 공급 | 11.2.3.2 | part4 **L263** |
| RB 폐회로 냉각수 (RBCLCW) | 11.3 | part4 L400 |
| 터빈건물 서비스수 (TBSW) | 11.4 | part4 L548 |
| TB 폐회로 냉각수 (TBCLCW) | 11.5 | part4 L672 |
| 계장공기 | 11.6 | part4 L801 |
| 운전 (기동·출력운전·정지) | 12.x | part4 L963~L1174 |
| BWR 형식별 차이 | 13.0 | part4 L1222 |

---

## 검증된 핵심 수치 (원문 ↔ 코드 대조 완료)

**판정 표기**: ✓ 일치 · **[규모]** 1.35배 환산 적용 · **[코드우선]** 매뉴얼보다 코드가 타당

### 노심 형상·물리

| 항목 | 매뉴얼 원문 | 코드 상수 |
|---|---|---|
| 유효연료 길이 | *"active fuel length of **150 inches**"* (2.2.2.1) + Fig 3.1-1 (358−208) | `CORE_HEIGHT` 3.81 m ✓ |
| 유효연료 상단 (TAF) | Fig 3.1-1 358 in (정상수위 554 기준 −196 in) | `TOP_OF_FUEL` −497.8 cm ✓ |
| 유효연료 하단 (BAF) | Fig 3.1-1 208 in (−346 in) | `BOTTOM_OF_FUEL` −878.8 cm ✓ |
| 다발 봉 개수 | *"54 standard fuel rods, eight fueled tie rods and **two water rods**"* (2.2.2.1.3) | `RODS_PER_BUNDLE` 62 · `WATER_RODS` 2 ✓ |
| 설계 LHGR | *"The design LHGR for 8x8 fuel is **13.4 kW/ft**"* (1.13.4) + DAEC TS 3.12 | `LHGR_LIMIT` 13.4 ✓ |
| MCPR 안전한계 | *"The MCPR safety limit is set at **1.07**"* (1.13.6) | `MCPR_SAFETY_LIMIT` 1.07 ✓ |
| MCPR 운전한계 | R-304B Fig 1.8-7 (**1.44**), 1.8.5.3 *"ranges from 1.2 to 1.5"* | `MCPR_OPERATING_LIMIT` 1.44 ✓ |
| MAPLHGR 저유량 | DAEC TS *"core flow ≤70% of rated … 95% of the limiting values"* | `MAPLHGR_LOW_FLOW` 0.95 ✓ |
| MAPLHGR 값 | **매뉴얼에 없음** (그래프로만 공개) | `MAPLHGR_LIMIT` 11.2 — **근거 약함** |
| 다발 내 국부첨두 | **설계값 없음** (Table 1.8-2 의 1.47~1.61 은 GEXL 시험 경계) | `LOCAL_PEAKING` 1.13 — **근거 없음** |
| 국부 연소도 | *"20 kW/ft at a local exposure of **40,000 Mwd/MT**"* (1.13.4) | 3주기 40,606 MWd/MT — +1.5% ✓ |
| 제논 반감기 | *"9.2 hr / 6.7 hr"* (1.12.7.1) | 9.14 h / 6.57 h — 정밀값 ✓ |
| 반응도계수 | α_V −1×10⁻³ Δk/k/%void 등 (1.12.5.4) | `VOID_COEF` −50 pcm — **[코드우선]** ※ |
| 지연중성자 6군 | (매뉴얼에 표 없음) | Keepin U-235 표준값과 완전 일치 ✓ |

> ※ **반응도계수는 매뉴얼을 따르지 않았다.** 매뉴얼이 스스로 *"**Approximate**
> numerical values"* 라 밝히고 세 값이 정확히 10의 거듭제곱이며, −100 pcm/%void 는
> **PWR 값**이다. BWR 설계범위 −70~−30 의 한가운데인 −50 을 유지했다.
> 상세는 `VERIFICATION.md` §4.3.2.

### 주증기·EHC

| 항목 | 매뉴얼 원문 | 코드 상수 |
|---|---|---|
| MSIV 구동방식 | *"air to open; air and/or spring to close"* (2.5.2.3) | fail-closed ✓ |
| 바이패스 용량 | *"Four bypass valves … up to 30% of rated steam flow"* (2.5.2.5) | `BYPASS_CAPACITY` 0.30 ✓ |
| 터빈 | *"1800 RPM, tandem compound"* (2.5.2.8) | `RATED_RPM` 1800 ✓ |
| TSV 폐쇄 스크램 문턱 | *"power above the capacity of the bypass valves"* (2.5.3.2) | `TSV_SCRAM_POWER` 0.30 ✓ |
| EHC 압력설정 | *"normally set at 920 psi"* (3.2.2.1) | 노심 7.03 MPa abs ≈ 990 psig dome ✓ |
| EHC 바이패스 | *"bypass valve demand = pressure control output − pressure/load LVG output"* (3.2.2.4) | `want = demand − turbine_valve` ✓ |
| 유량제한기 | *"limit steam flow to less than **200%** of rated steam flow"* (2.5.2.2) | 파단유량 상한 ✓ |
| MSIV 격리 조건 | Table 4.4-1 — 고방사선·터널고온·고유량(RUN)·저압·L2·수동 | 구성 일치 ✓ (**수치 설정치는 표에 없다**) |
| SRV 대수 | **매뉴얼 자기모순** — 11/7 (2.5.2.1 2회, 10.2 2회) ↔ 13/6 (2.5.4.11 한 줄) | 13 / ADS 6 ✓ — **브라운스페리 기준** ※ |

> ※ **SRV 대수는 매뉴얼보다 발전소를 봐야 한다.** 매뉴얼은 11/7 쪽이 우세하지만
> (2.5.2.1 에 두 번, 10.2 에 두 번 — 13 은 2.5.4.11 상호참조 한 줄뿐),
> R-304B Table 1.8-1 이 3293 MWt BWR/4 를 **브라운스페리**로 밝혔고 브라운스페리는
> SRV 13 대에 ADS 6 대다. 매뉴얼의 11/7 은 더 작은 기준 발전소 값이다(§규모 보정).
> 코드의 13/6 이 맞다. 상세는 `VERIFICATION.md` §12.2.

### 보호계통·반응도제어

| 항목 | 매뉴얼 원문 | 코드 상수 |
|---|---|---|
| APRM 인출차단 | *".66W + 42%"* (Table 7.1-1 = Table 5.4-1 "APRM High") | `APRM_FLOW_SLOPE` 0.66 / `APRM_BLOCK_BIAS` 42 ✓ |
| APRM 차단 상한 | *"Flow Converter Hi 108%"* (Table 7.1-1) | `APRM_BLOCK_CLAMP` 108 ✓ |
| APRM 유량바이어스 스크램 | *".66(W) + 51%, 113.5% Max."* (**Table 5.4-1**, part2:3366) | `APRM_SCRAM_BIAS` 51 / `APRM_SCRAM_CLAMP` 113.5 ✓ |
| APRM 고정출력 스크램 | *"Fixed Power 118% — Rod Block & Scram"* (Table 5.4-1) | 미구현 — 113.5% 클램프 아래라 도달 불가 |
| RBM 인출차단 | *".66W + 41% (set high)"* (Table 7.1-1) | `RBM_SLOPE` 0.66 / `RBM_BIAS` 41 ✓ |

> **함정 — Table 7.1-1 에는 스크램이 없다.**
> 7.1-1 의 제목은 *"Rod Withdrawal Blocks and Setpoints"* 로 **인출차단만** 싣는다.
> 스크램 설정치는 **Table 5.4-1**(`part2:3366`) 에 따로 있고 값이 다르다.
>
> ```
> APRM High           .66(W) + 42%   108% Max.     Rod Block
> APRM High-High      .66(W) + 51%   113.5% Max.   Scram
> APRM Fixed Power    118%                         Rod Block & Scram
> ```
>
> 한 표 안에서 **행의 '동작' 열을 먼저 확인**한다. 숫자만 맞춰 보면 차단 값을
> 스크램 자리에 넣어도 MATCH 가 나온다 (실제로 그렇게 들어가 있었다 — VERIFICATION §13.1).
| RBM Inop | *"Fail to null / More than one rod selected"* | `rbm_null()` ✓ |
| EOC-RPT | *"main turbine trip or load rejection, if greater than 30% power"* (7.2.3.2) | `RPT_POWER` 0.30 ✓ |
| ATWS-RPT | *"1120 psig or low-low level"* (7.2.3.2) | `ATWS_RPT_PRESSURE` 7.82 MPa abs, `LEVEL_2` ✓ |

### SLC — 전 항목 일치 (유일하게 불일치 0)

| 항목 | 매뉴얼 원문 | 코드 상수 |
|---|---|---|
| 탱크 | *"4,850 gallons"* (7.4.2.1) | `SLC_TANK_VOLUME` 18.36 m³ ✓ |
| 펌프 | *"two 100% capacity … greater than 39 gpm"* (7.4.2.2) | `SLC_PUMP_FLOW` 2.7 kg/s ×2 ✓ |
| 용액 | *"13 weight percent sodium pentaborate"* | `SLC_BORON_FRACTION` 0.0238 ✓ |
| 기동 동작 | *"starts a single pump, fires both explosive valves, and isolates RWCU"* (7.4.3.1) | `arm_slc()` ✓ |
| 정지시간 | *"one to two hour … **not a backup for scram**"* (7.4.3.1) | 실측 57분 ✓ |
| 주입점 | *"beneath the core plate into … jet pump diffuser outlets"* | `SLC_MIX_TAU` ✓ |

### ECCS

| 항목 | 매뉴얼 원문 | 코드 상수 |
|---|---|---|
| **LPCI 유량** | *"10,000 gpm"* (10.4.2.2) | `LPCI_FLOW` 630 kg/s ✓ — **양쪽 일치(규모 무관)** |
| HPCI 유량 | 4000 gpm (2436 MWt 급) | `HPCI_FLOW` 268 kg/s = 4250 gpm — **[규모]** |
| RCIC 유량 | 400 gpm (2436 MWt 급) | `RCIC_FLOW` 38 kg/s = 600 gpm — **[규모]** |
| CS 유량 | 4725 gpm (2436 MWt 급) | `CS_FLOW` 400 kg/s = 6340 gpm — **[규모]** |
| CS 계열 수 | *"There are a total of **two** core spray pumps"* (10.3) | `CS_LOOPS` 2 ✓ |
| LPCI 계열 수 | *"The **A and C** pumps … the **B and D** pumps"* (10.4.2.2) | `LPCI_LOOPS` 4 ✓ |
| ADS 타이머 | **105초** (10.2.3.1 본문 5회 + Fig 10.2-1) | `ADS_DELAY` 105 ✓ |
| HPCI 증기압 하한 | ~100 psig 정지밸브 (10.0.7) | `HPCI_P_MIN` 0.79 MPa ✓ |
| HPCI 증기압 상한 | 1150 psig (10.1.2) | `HPCI_P_MAX` 8.03 MPa ✓ |
| RCIC 증기압 하한 | 150 psig (2.7.1) | `RCIC_P_MIN` 1.14 MPa ✓ |
| HPCI 터빈트립 | ①배기압고 ②과속 ③흡입압저 ④격리신호 ⑤고수위 (10.1.3.4) | 5조건 전부 ✓ |
| HPCI 재기동 | *"if vessel level subsequently decreases to Level 2 … automatically reinitiate"* | seal-in ✓ |
| HPCI/RCIC 증기원 | RCIC=A 증기관, HPCI=B 증기관 (2.5.4.3–4) | MSIV 상류 ✓ |
| ECCS 모선 배분 | **Table 9.2-1 ↔ 10.4.2.2 — 매뉴얼 자기모순** | 부하표를 따름 (A→101, B→102, C·D→103) |
| CS/LPCI 체결수두 | **매뉴얼에 없음** (정격토출압만: CS 274 · RHR 136 psig) | `CS_P_MAX`·`LPCI_P_MAX` 1.90 MPa — **근거 없음** |

### 격납용기 (Table 4.1-1)

| 항목 | 매뉴얼 원문 | 코드 상수 |
|---|---|---|
| 드라이웰 자유체적 | 159,000 ft³ = 4502 m³ | `DRYWELL_VOLUME` 4500 ✓ |
| 압력억제실 기상부 | 119,000 ft³ = 3370 m³ | `WETWELL_GAS_VOLUME` 3400 ✓ |
| **억제수조 수량** | *"Water Volume **135,000 ft³**"* | `POOL_VOLUME` 3822.8 m³ ✓ |
| **설계압력** | *"Maximum Internal Design Pressure **62 psig**"* | `CONT_DESIGN_PRESSURE` 528.8 kPa ✓ |
| 외압 설계값 | *"Maximum External Design Pressure **2 psig**"* | **모델에 없음** (내압만 판정) |
| 설계온도 | **281 °F (138.3℃)** | 모델은 압력만 판정 — 수조 138℃ 초과 시 이미 설계조건 밖 ⚠ |
| 진공파괴밸브 | *"0.5 psi differential"* (4.1.2.3) | `VACUUM_BREAKER_DP` 3.4 kPa ✓ |
| 드라이웰 정상온도 | *"less than 150°F"* (65.6℃) | `DW_TEMP0` 57℃ ✓ (범위 내) |

### 전기 — 수치 오류 0건

| 항목 | 매뉴얼 원문 | 코드 상수 |
|---|---|---|
| 비상모선 | *"red-101, blue-102, orange-103 / no bus ties"* (9.2.2.1) | `BUS_NAMES` ✓ |
| 디젤 정격 | *"continuous rating … 3500 KW"* (9.2.2.2) | `DG_RATING_KW` 3500 ✓ |
| 디젤 기동시간 | *"less than 10 seconds"* (9.2.2.2) | `DG_START_TIME` 10 ✓ |
| 디젤 자동기동 | ①드라이웰 고압 ②수위 L1 ③4160V 모선 전압상실 ④해당 보드 전압상실 (9.2.3.2) | 4신호 ✓ |
| 디젤 냉각 | *"water cooled"* + RBSW 가 엔진냉각 공급 (11.2.3.2) | `DG_COOLING_LOOP` ✓ |
| 모선 부하 | 디젤 3500 kW 이내 | 최악 3054~3100 kW ✓ |
| UPS 지속 | *"minimum of two hours in the event of a total loss of AC power"* (9.3.1.4) | 축전지 8h / RCIC 부하 약 6h ✓ |
| 축전지 지속시간 | **매뉴얼에 수치 없음** | `DC_ENDURANCE_H` 8.0 — 후쿠시마 1호기 기준 |

### 2차계통·서비스수

| 항목 | 매뉴얼 원문 | 코드 상수 |
|---|---|---|
| 발전소 기준 | Table 1.5-1 *"3293 MWT · 1065 MWE · 764 다발 · 185 제어봉"* | `RATED_MW` 3293 ✓ |
| 복수펌프 | *"Two motor driven condensate pumps, 50% each"* (2.6.2.2) ※ | `CONDENSATE_PUMPS` 2 ✓ |
| 급수펌프 | *"driven by variable speed steam turbines"* (2.6.2.7) | `FEED_PUMPS` 2 ✓ |
| 순환수펌프 대수 | *"four circulating water pumps in operation"* (11.1.3) | `CIRC_PUMPS` 4 ✓ |
| 급수가열기 | *"one drain cooler and three low pressure heaters"* (2.6.2.6) + *"two heaters per string"* (2.6.2.8) | DC + LP×3 + HP×2 ✓ |
| 최종 급수온도 | 420°F (Figure 2.0-2) | 215.6℃ ✓ (2차계통이 계산) |
| 순환수펌프 유량 | *"Each pump … 143,400 gpm"* (11.1.2.1) = 9,047 kg/s | `CIRC_EACH` **12,230** — **[규모]** |
| 순환수펌프 전력 | *"Each motor is rated for 1,500 hp"* (11.1.2.1) = 1.12 MW | `AUX_CIRC_EACH` **1.51** — **[규모]** |
| **RBSW 펌프 유량** | *"Four service water pumps … 8600 gpm"* (11.2.2) = 543 kg/s | `RBSW_PUMP_FLOW` **734** — **[규모]** |
| **RBSW 펌프 전력** | *"… 450 Hp"* (11.2.2) = 336 kW | `RBSW_PUMP_KW` **454** — **[규모]** |
| RBSW 루프 | *"two loops … one pump per loop (A or C and B or D)"* (11.2.3.1) | `RBSW_LOOP` (0,1,0,1) ✓ |
| RBSW 헤더격리 | *"automatically close during LOCA or loss of emergency bus voltage"* (11.2.1) | `rbsw_isolated` ✓ |

> ※ **복수펌프 대수도 매뉴얼이 자기모순이다.** 2.6 개요는 *"removed by **three**
> condensate pumps"*, 2.6.2.2 는 *"**Two** motor driven condensate pumps, with a
> capacity of **50% each**"*. 2대×50% = 100% 로 앞뒤가 맞는 쪽은 2.6.2.2 다.

> **RBSW 두 값은 예전에 "문헌과 정확히 일치"라고 적어 뒀던 것이다.** 일치한 건
> 맞지만 **그 문헌이 더 작은 발전소**였다. 지금은 1.35배 환산값이 들어가 있다.

---

## 원자로 수위 설정치 (3.1.3.1) — 전 항목 일치

매뉴얼은 계기영점 기준(내부 바닥머리 0in, 계기영점 517in, 정상수위 554in).
정상수위(+37")를 0 으로 옮겨 환산한다.

| 레벨 | 매뉴얼 | 환산 | 코드 | 차이 |
|---|---|---|---|---|
| Level 8 | **+56.5"** | +49.5 cm | `LEVEL_8` = +49.5 | ✓ |
| Level 7 (경보) | +40.5" | +8.9 cm | `LEVEL_7` = +8.9 | ✓ |
| Level 5 (정상) | ~+37" | 0 | `LEVEL_NORMAL` = 0 | ✓ |
| Level 4 (경보) | +33.5" | −8.9 cm | `LEVEL_4` = −8.9 | ✓ |
| Level 3 (스크램) | **+12.5"** | −62.2 cm | `SCRAM_LEVEL` = −62.0 | +0.2 cm |
| Level 2 | **−38"** | −190.5 cm | `LEVEL_2` = −191.0 | −0.5 cm |
| Level 1 | **−132.5"** | −430.5 cm | `LEVEL_1` = −430.5 | ✓ |

> 예전에는 L8 이 +43.0(+54"), L1 이 −422.0(−129") 로 매뉴얼과 달랐다. 지금은 전부
> 매뉴얼 값이다. L7·L4 는 원래 없던 것을 신설했다.

**설정치의 근거 (3.1.3.1)**

- **L8** (3.1.3.1.1): **셋**을 트립한다 — 주터빈(습분 캐리오버 방지) · 급수펌프터빈
  (과충수 방지) · RCIC/HPCI 터빈(증기관 침수 방지). **셋 다 구현됨** (주터빈·급수펌프
  터빈은 래치라 운전원이 복구).
- **L4** (3.1.3.1.4): 급수펌프 트립과 겹치면 **재순환 런백**. **구현됨**
  (`RECIRC_RUNBACK` 0.45, 래치).
- **L3**: 건조기 실 스커트 아래로 증기가 새는 것을 막고, 유효연료 상단까지 여유
  냉각재를 남긴다.
- **L2** (3.1.3.1.6): 스크램 후 보이드 붕괴에 의한 수위 강하로는 안 걸리게 낮게,
  동시에 RCIC 가 L1 도달을 막을 수 있게 높게. **ATWS-RPT 도 이 신호.**
- **L1** (3.1.3.1.7): **유효연료 상단보다 충분히 높아** ECCS 가 동작할 시간을 확보.
  → 이 요구가 TAF 오류를 잡아낸 근거다. 예전 TAF(−368)로는 L1 이 TAF 아래에 있어
  ECCS 가 노심 17% 노출 후에야 걸렸다.

---

## 인용 정정 (시뮬레이터 문서 → 매뉴얼)

| 시뮬레이터 문서의 인용 | 실제 | 비고 |
|---|---|---|
| "NRC 10.13" (HPCI 자동트립) | **10.1.3.4** | 내용(5조건 중 ⑤ 고수위)은 정확. OCR이 `10.1.3.2`를 `10.13.2`로 뭉갠 데서 온 오기 |
| "NRC 3.2.2.2" (EHC 바이패스) | **3.2.2.4** | 3.2.2.2는 Load Control Unit. 바이패스 지령식은 Valve Control Unit |
| "NRC 2.5.3.2" (EOC-RPT) | 맞음 | 다만 상세는 **7.2.3.2** 에 있음 (30% · 1120 psig · 차단기) |
| "NRC 4.1.3.3" (퍼지) | 맞음 | Nitrogen Inerting. 운전 중 퍼지는 4.1.3.6/4.1.3.7 도 참고 |
| "NRC 11.2" (RBSW) | 맞음 | |
| "NRC 9.2 Table 9.2-1" | 맞음 | |
| "NRC 7.4" (SLC) | 맞음 | |
| "NRC Table 7.1-1" | 맞음 | |
| "NRC 2.5.2.1 / 2.5.2.3" | 맞음 | |

---

## 매뉴얼에 있으나 모델에 없는 것 — 찾을 때 쓸 위치

| 미구현 항목 | 절 | 위치 | 중요도 |
|---|---|---|---|
| ~~열적 제한 MCPR·LHGR·APLHGR~~ | 1.13 | part1 L1741~ | **완료** (§5.21) |
| └ 노출 의존 한계곡선 · 출력/유량 의존 OLMCPR | 1.13.4/5 · R-304B Fig 1.8-9/10 | — | 중 |
| ~~사마륨-149~~ | 1.12.7.2 | part1 L1666 | **완료** (§5.22) |
| **복수 부스터펌프** | 2.6.2.5 | part1 **L5030** | 높음 |
| **복수 여과·탈염기** | 2.6.2.4 | part1 **L5039** | 높음 |
| **외부전원 이중화·고속절체** | 9.2.3.3 | part3 **L2735** | 높음 |
| 120 VAC 계통 전체 | 9.3 | part3 L2843 / L2861 | 중 |
| 직류 division 분할 | 9.4 | part3 L3033 | 중 |
| 급수가열기 3계열 병렬 | 2.6.2.6/2.6.2.8 | part1 **L5039** | 중 |
| 습분분리기·재열기 | 2.5.2.8 계열 | part1 L4599~ | 중 |
| 드라이웰/압력억제실 차압제어 | 4.1.3.5 | part2 L1113 | 중 |
| 격납 외압 한계 (2 psig) | Table 4.1-1 | part2 L1472 | 낮음 |
| CAD 계통 | 4.x | part2 L1100~ | 낮음 |
| SRM/IRM (24 VDC 포함) | 5.1–5.2 | part2 L2532~ | 낮음 |
| 순환수 진공프라이밍·수막 | 11.1.2 | part4 L100~ | 낮음 |

전체 목록과 근거는 `VERIFICATION.md` §9.

---

## 참고 — 냉각탑은 의도적으로 다르다

RBSW 방출처가 *"Long Island Sound"* 로 나온다 → **해수 일과통과(open-cycle) 부지** 기준.
시뮬레이터는 내륙 부지를 가정해 냉각탑 순환 방식으로 바꿨다(CHANGES.md §3에 명시).
따라서 `RBSW_INTAKE_TEMP` · `TOWER_*` · `WET_BULB` · `BASIN_*` 는 **매뉴얼 대조 대상이
아니다**.

## 참고 — 검증할 때 밟았던 함정

1. **부분 일치는 근거가 아니다.** 모든 계통에서 *일부* 상수만 매뉴얼과 소수점까지
   맞고 나머지는 다른 출처였다 (수위: L2·L3만 / ECCS: LPCI만 / 격납: 기상부만 /
   11장: RBSW만). 몇 개가 맞는다고 표 전체를 적용한 것으로 보면 안 된다.
2. **매뉴얼이 자기모순인 곳이 셋 있다** — SRV 대수(11/7 ↔ 13/6), ECCS 모선 배분
   (Table 9.2-1 ↔ 10.4.2.2), 복수펌프 대수(개요 "three" ↔ 2.6.2.2 "Two … 50% each").
   멀쩡한 코드를 고칠 뻔했다.
   → **"계통을 소유한 장을 따른다" 는 잘못된 기준이었다.** SRV 에서는 소유 장(2.5)이
   오히려 11 이라고 말한다. 옳은 기준은 **"이 숫자가 어느 발전소를 말하는가"** 다
   (SRV 13 은 브라운스페리 = 이 모델의 기준). 그 다음이 형식 문서(부하표)와
   내부 정합성(2대×50% = 100%)이다.
3. **주석은 근거가 아니다.** ADS 무장 조건 주석은 코드가 안 하는 일을 적고 있었고,
   `HPCI_FLOW` 주석의 gpm 값은 매뉴얼과 달랐다.
4. **pickle 이 상수 변경을 삼킨다.** `POOL_MASS` 는 인스턴스 상태로 복사되므로
   `aged.pkl` 을 다시 만들지 않으면 옛 값이 살아 있다. 상수를 바꿨는데 결과가 안
   변하면 "영향 없음" 이 아니라 "반영 안 됨" 을 먼저 의심할 것.
