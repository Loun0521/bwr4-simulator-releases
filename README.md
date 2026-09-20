# BWR-4 Simulator — 공식 베타 배포

이 저장소는 BWR-4 Simulator의 **공식 실행 배포본과 사용자 문서**를 제공한다.
시뮬레이터 원본 소스 코드는 여기에 포함하지 않는다.

## 받는 곳

[최신 pre-release](https://github.com/Loun0521/bwr4-simulator-releases/releases)에서
운영체제와 CPU에 맞는 파일을 받으면 된다.

| 환경 | 실행 배포 파일 |
|---|---|
| Windows 10/11, Intel·AMD 64비트 | `Windows-x64.zip` |
| Apple Silicon Mac | `macOS-arm64.zip` |
| Intel Mac | `macOS-x64.zip` |
| Ubuntu 22.04+ 호환 Linux, Intel·AMD 64비트 | `Linux-x64.tar.gz` |
| Ubuntu 24.04+ 호환 Linux, ARM64 | `Linux-arm64.tar.gz` |

압축 파일과 같은 이름의 `.sha256` 파일로 무결성을 확인할 수 있다. `beta.2`
이후에는 압축을 완전히 푼 뒤 `OPEN_DOCUMENTATION.html`을 열면 기본 브라우저가
버전이 고정된 GitHub 초보자 안내를 연다.

## GitHub에서 문서 읽기

파일을 내려받지 않고 다음 문서를 GitHub 화면에서 바로 읽을 수 있다.

| 버전 | 초보자 안내 | 물리 | 검증 | 공식 매뉴얼 자료 |
|---|---|---|---|---|
| `v0.2.0-beta.2` | [처음 시작하기](docs/v0.2.0-beta.2/START_HERE.md) | [물리 모델](docs/v0.2.0-beta.2/DOCUMENTATION/PHYSICS.md) | [검증 보고서](docs/v0.2.0-beta.2/DOCUMENTATION/VERIFICATION.md) | [공식 매뉴얼 색인](docs/v0.2.0-beta.2/DOCUMENTATION/OFFICIAL_MANUAL/INDEX.md) |
| `v0.2.0-beta.1` | [처음 시작하기](docs/v0.2.0-beta.1/START_HERE.md) | [물리 모델](docs/v0.2.0-beta.1/DOCUMENTATION/PHYSICS.md) | [검증 보고서](docs/v0.2.0-beta.1/DOCUMENTATION/VERIFICATION.md) | [공식 매뉴얼 색인](docs/v0.2.0-beta.1/DOCUMENTATION/OFFICIAL_MANUAL/INDEX.md) |

GitHub가 릴리스에 자동으로 표시하는 `Source code (zip)`과 `Source code (tar.gz)`는
이 배포 저장소의 README·라이선스·공개 문서를 묶은 파일이다. 시뮬레이터 실행
파일이나 Python 원본 소스는 들어 있지 않다. 실행하려면 위 표의 운영체제별
배포 파일을 받아야 한다.

## 사용 조건

배포본은 개인·비상업적 베타 평가와 교육 목적으로만 실행할 수 있다. 파일을
다른 곳에 다시 올리거나, 미러링·판매하거나, 수정·재포장해 배포할 수 없다.
다른 테스터에게는 이 공식 저장소의 release 주소를 공유해 줘. 자세한 조건은
[LICENSE.md](LICENSE.md)에 있다.

이 프로그램은 교육용 베타다. 실제 발전소 운전, 운전원 자격, 안전해석, 인허가
판단, 비상계획, 공학 판단이나 실제 설비 절차에 사용하면 안 된다.
