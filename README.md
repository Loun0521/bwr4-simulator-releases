# BWR-4 Simulator — 공식 베타 배포

이 저장소는 BWR-4 Simulator의 **실행 배포본만** 제공한다. 시뮬레이터 원본
소스 코드는 비공개 저장소에서 관리하며 여기에 포함하지 않는다.

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

압축 파일과 같은 이름의 `.sha256` 파일로 무결성을 확인할 수 있다. 압축을
완전히 푼 뒤 포함된 `START_HERE.md`부터 읽어 줘. GitHub가 자동으로 표시하는
`Source code (zip)`과 `Source code (tar.gz)`에는 이 README와 라이선스만 있고,
시뮬레이터 실행 파일이나 소스 코드는 없다.

## 사용 조건

배포본은 개인·비상업적 베타 평가와 교육 목적으로만 실행할 수 있다. 파일을
다른 곳에 다시 올리거나, 미러링·판매하거나, 수정·재포장해 배포할 수 없다.
다른 테스터에게는 이 공식 저장소의 release 주소를 공유해 줘. 자세한 조건은
[LICENSE.md](LICENSE.md)에 있다.

이 프로그램은 교육용 베타다. 실제 발전소 운전, 운전원 자격, 안전해석, 인허가
판단, 비상계획, 공학 판단이나 실제 설비 절차에 사용하면 안 된다.

