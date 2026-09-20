# BWR-4 Simulator v0.2.0-beta.1 — 처음 시작하기

이 배포본은 기능과 화면을 확인하기 위한 **베타 테스트용 교육 시뮬레이터**다.
발전소 운전, 운전원 자격, 안전해석, 인허가 판단, 비상계획이나 실제 설비 절차에
사용하면 안 된다.

공식 배포처는 다음 GitHub Releases 페이지 하나다.

https://github.com/Loun0521/bwr4-simulator-releases/releases

개인·비상업적 베타 평가를 위해 실행할 수 있다. 내려받은 압축 파일이나 실행
파일을 다른 사람에게 직접 보내거나, 다른 사이트에 다시 올리거나, 수정·재포장해
배포하면 안 된다. 다른 테스터에게는 위 공식 배포 페이지 주소를 보내 줘. 자세한
조건은 압축 파일의 `LICENSE.md`에 있다.

## 내 컴퓨터에 맞는 파일

| 운영체제 | CPU | 받을 파일 |
|---|---|---|
| Windows 10/11 | Intel·AMD 64비트 | `Windows-x64.zip` |
| macOS | Apple Silicon(M1 이상) | `macOS-arm64.zip` |
| macOS | Intel Mac | `macOS-x64.zip` |
| Ubuntu 22.04+ 호환 Linux | Intel·AMD 64비트 | `Linux-x64.tar.gz` |
| Ubuntu 24.04+ 호환 Linux | ARM64 | `Linux-arm64.tar.gz` |

CPU를 모르겠다면 macOS는 화면 왼쪽 위 **Apple 메뉴 → 이 Mac에 관하여**에서
`칩`이 Apple M 계열이면 arm64, `프로세서`가 Intel이면 x64를 받으면 된다.

## Windows 실행

1. ZIP을 새 폴더에 **완전히 압축 해제**한다.
2. 폴더 안의 `BWR4-Simulator.exe`를 실행한다.
3. Python과 Git은 필요하지 않다.
4. 이번 베타는 코드 서명 인증서가 없어 Windows가 `알 수 없는 게시자` 경고를
   표시할 수 있다. 파일 이름과 함께 제공된 SHA-256을 공식 배포 페이지의 값과
   먼저 대조한다.

## macOS 실행

1. 자신의 CPU에 맞는 ZIP을 완전히 압축 해제한다.
2. `BWR4-Simulator.app`을 Control-클릭하고 **열기**를 선택한다.
3. Apple 공증을 받지 않은 베타이므로 처음 한 번 보안 경고가 나타날 수 있다.
4. Python과 Git은 필요하지 않다.

## Linux 실행

압축을 푼 폴더의 터미널에서 다음처럼 실행한다.

```bash
tar -xzf BWR4-Simulator-v0.2.0-beta.1-Linux-x64.tar.gz
cd BWR4-Simulator-v0.2.0-beta.1-Linux-x64
./BWR4-Simulator
```

ARM64에서는 파일과 폴더 이름의 `x64`를 `arm64`로 바꾼다. 그래픽 데스크톱과
X11 또는 호환 디스플레이 환경이 필요하다. 다른 배포판은 glibc와 시스템 GUI
라이브러리 차이 때문에 이번 베타의 지원 범위 밖이다.

## 사용자 데이터와 오류 기록

`5일 운전` 보관분과 오류 기록은 프로그램 폴더 밖에 저장된다.

| 운영체제 | 사용자 데이터 폴더 |
|---|---|
| Windows | `%LOCALAPPDATA%\BWR4Simulator` |
| macOS | `~/Library/Application Support/BWR4Simulator` |
| Linux | `$XDG_DATA_HOME/BWR4Simulator` 또는 `~/.local/share/BWR4Simulator` |

오류 기록 파일은 위 폴더 아래의 `logs/startup-error.log`다. 저장 상태에는 Python
pickle이 포함되므로 다른 사람이 보낸 상태 파일로 바꾸지 않는 것이 좋다.

## 포함 문서

- `START_HERE.md` — 이 초보자용 실행 안내
- `DOCUMENTATION/PHYSICS.md` — 시뮬레이터 물리와 가정
- `DOCUMENTATION/VERIFICATION.md` — 구현값·공식 근거·검증 범위
- `DOCUMENTATION/OFFICIAL_MANUAL/` — R-104B/R-304B 검색용 추출 텍스트와 원문 링크
- `LICENSE.md`와 `THIRD_PARTY_LICENSES/` — 사용 조건과 필수 제3자 고지

개발 변경 이력, 구현 계획, 테스트 내부 보고서와 저장소 관리 문서는 실행 배포본에
넣지 않는다.

## 이번 베타에서 확인할 것

- 압축 해제 후 별도 설치 없이 시작하는지
- 화면 배율 100%, 125%, 150%에서 글자와 조작부가 잘리지 않는지
- `5일 운전` 완료 후 종료·재실행했을 때 같은 노심의 보관분을 읽는지
- 도움말, 계통도, 전기계통도와 하단 2차 격납 조작부가 열리는지
- 오류가 나면 사용한 운영체제·CPU·화면 배율·조작 순서와 오류 기록
