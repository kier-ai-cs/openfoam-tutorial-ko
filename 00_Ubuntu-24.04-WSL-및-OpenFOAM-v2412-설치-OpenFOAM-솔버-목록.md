# 예제 0. Ubuntu 24.04 (WSL) 및 OpenFOAM v2412 설치, OpenFOAM 내장 솔버 목록

**참고**:

- 이 자료는 [CC BY-NC-SA 3.0](https://creativecommons.org/licenses/by-nc-sa/3.0/) 라이선스 하에 배포된다.
- 이 자료는 ESI Group, ESI-OpenCFD 또는 OpenFOAM Foundation에 의해 승인된 자료가 아니다.

## 목차

1. WSL을 이용한 Ubuntu 24.04 설치 (Windows 11)
   - 1.1 WSL 및 Ubuntu 24.04 설치
   - 1.2 WSL 버전 확인
   - 1.3 Windows 파일 시스템 접근
2. OpenFOAM v2412 설치 (Ubuntu 24.04)
   - 2.1 필수 패키지 설치
   - 2.2 소스팩 다운로드 및 압축 해제
   - 2.3 권한 변경 및 컴파일
   - 2.4 환경 변수 설정
   - 2.5 설치 확인
   - 2.6 예제 파일 경로 확인
   - 2.7 Alias
3. OpenFOAM v2412 내장 솔버 목록
   - 3.1 기본 해석
   - 3.2 비압축성 유동 해석
   - 3.3 압축성 유동 해석
   - 3.4 열전달 해석
   - 3.5 연소 해석
   - 3.6 다상 유동 해석
   - 3.7 입자 거동 해석
   - 3.8 표면/막 해석
   - 3.9 전자기장 해석
   - 3.10 분자 동역학 해석
   - 3.11 직접 수치 모사
   - 3.12 응력-변형률 해석
   - 3.13 금융 해석

## 1. WSL을 이용한 Ubuntu 24.04 설치 (Windows 11)

Windows 11에서 WSL(Windows Subsystem for Linux)을 이용하여 Ubuntu를 설치하고, 그 위에서 OpenFOAM을 설치, 실행한다.

### 1.1 WSL 및 Ubuntu 24.04 설치

관리자 권한으로 명령 프롬프트 또는 PowerShell을 열고 아래 명령어를 입력하여 WSL를 활성화하고 리눅스 배포판 Ubuntu 24.04를 설치한다.

```bash
wsl --install -d Ubuntu-24.04
```

설치가 완료되면 사용자 이름과 비밀번호를 설정하라는 메시지가 표시된다. 사용자 이름과 비밀번호를 설정하면 자동으로 Ubuntu 터미널이 열린다.

Ubuntu 터미널에서 아래 명령어를 입력하고 비밀번호를 입력하여 시스템 패키지를 최신 상태로 업데이트한다.

```bash
sudo apt update && sudo apt upgrade -y
```

완료되면 Ubuntu 터미널을 종료하고, 시작 메뉴에서 "Ubuntu 24.04"를 검색하여 재실행한다.

### 1.2 WSL 버전 확인

명령 프롬프트에 아래 명령어를 입력하여 WSL 버전을 확인한다.

```bash
wsl -l -v
```

```text
  NAME            STATE           VERSION
* Ubuntu-24.04    Running         2
```

출력 결과에서 VERSION 열이 2인지 확인한다. 만약 1인 경우, 아래 명령어를 입력하여 WSL2 버전으로 변경한다.

```bash
wsl --set-version Ubuntu-24.04 2
```

### 1.3 Windows 파일 시스템 접근

WSL에서 Windows의 파일 시스템은 /mnt/ 하위에 마운트된다. 예를 들어, C 드라이브는 /mnt/c/에서 접근할 수 있다.

```bash
ls /mnt/c/Users/
```

> **Note:** OpenFOAM의 성능을 위해 케이스 파일은 Windows 파일 시스템(/mnt/c/)이 아닌 WSL의 Linux 파일 시스템(/home/사용자이름/)에 저장하는 것을 권장한다. WSL의 Linux 파일 시스템은 Windows 탐색기에서 주소창에 `\\wsl$`을 입력하여 접근할 수 있다.

## 2. OpenFOAM v2412 설치 (Ubuntu 24.04)

WSL2 Ubuntu 24.04에서 OpenFOAM v2412를 소스 컴파일하여 설치하는 방법을 설명한다.

### 2.1 필수 패키지 설치

Prerequisite 패키지를 설치한다.

```bash
sudo apt install build-essential autoconf autotools-dev cmake gawk gnuplot \
  flex libfl-dev libreadline-dev zlib1g-dev openmpi-bin libopenmpi-dev libncurses-dev \
  libgmp-dev libmpfr-dev libmpc-dev libfftw3-dev libscotch-dev libptscotch-dev \
  libboost-system-dev libboost-thread-dev libcgal-dev ninja-build \
  qtbase5-dev qttools5-dev qttools5-dev-tools \
  libqt5x11extras5-dev libqt5svg5-dev libqt5help5 \
  qtxmlpatterns5-dev-tools \
  libxt-dev libxext-dev libx11-dev \
  mesa-common-dev libgl1-mesa-dev libglu1-mesa-dev \
  libgl-dev python3-dev paraview -y
```

### 2.2 소스팩 다운로드 및 압축 해제

OpenFOAM v2412와 ThirdParty 소스팩을 /opt/에 다운로드하고 압축을 해제한다.

```bash
cd /opt
```

```bash
sudo wget -O OpenFOAM-v2412.tgz https://dl.openfoam.com/source/v2412/OpenFOAM-v2412.tgz
```

```bash
sudo wget -O ThirdParty-v2412.tgz https://dl.openfoam.com/source/v2412/ThirdParty-v2412.tgz
```

```bash
sudo tar xzf OpenFOAM-v2412.tgz && sudo tar xzf ThirdParty-v2412.tgz
```

### 2.3 권한 변경 및 컴파일

압축 해제된 디렉토리의 소유권을 루트에서 현재 사용자로 변경한 후 컴파일한다.

```bash
sudo chown -R $USER:$USER /opt/OpenFOAM-v2412 /opt/ThirdParty-v2412
```

```bash
source /opt/OpenFOAM-v2412/etc/bashrc
```

> **Note:** 여기서, 아래 경고는 무시한다.
>
>```text
>No completions for /opt/OpenFOAM-v2412/platforms/linux64GccDPInt32Opt/bin
>[ignore if OpenFOAM is not yet compiled]
>```

```bash
cd $WM_PROJECT_DIR
```

```bash
./Allwmake -j $(nproc) -s -l
```

> **Note:** `-j $(nproc)`는 사용 가능한 모든 CPU 코어를 사용하여 병렬 컴파일한다. `-s` 간략 출력 모드이며, `-l`는 빌드 로그를 파일로 기록한다.

> **Note:** 컴파일에는 30분에서 1시간 이상 소요될 수 있다.

### 2.4 환경 변수 설정

OpenFOAM의 환경 변수를 셸 시작 시 자동으로 불러오도록 설정한다.

```bash
echo "source /opt/OpenFOAM-v2412/etc/bashrc" >> ~/.bashrc
```

```bash
source ~/.bashrc
```

### 2.5 설치 확인

설치 및 환경 변수 설정이 정상적으로 완료되었는지 확인한다.

```bash
simpleFoam -help
```

솔버의 사용법과 옵션이 출력되면 설치가 정상적으로 완료된 것이다.

### 2.6 예제 파일 경로 확인

OpenFOAM의 예제 파일은 아래 경로에 저장되어 있다.

```bash
echo $FOAM_TUTORIALS
```

예제 파일을 사용자 작업 디렉토리로 복사하여 사용한다.

```bash
mkdir -p $FOAM_RUN
```

```bash
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/elbow $FOAM_RUN/
```

### 2.7 Alias

OpenFOAM이 정상적으로 설치되면, 아래의 alias가 함께 정의된다.

```text
alias run='cd ${FOAM_RUN:-${WM_PROJECT_USER_DIR:?}/run}'
alias tut='cd ${FOAM_TUTORIALS:-${WM_PROJECT_DIR:?}/tutorials}'
alias src='cd ${WM_PROJECT_DIR:?}/src'
```

## 3. OpenFOAM v2412 내장 솔버 목록

### 3.1 기본 해석

| 솔버 | 설명 |
| :--- | :--- |
| `laplacianFoam` | 스칼라 물리량에 대한 라플라스 방정식 해석 |
| `overLaplacianDyMFoam` | 스칼라 물리량에 대한 라플라스 방정식 해석. 이동 격자. overset 격자 |
| `potentialFoam` | 퍼텐셜 유동 해석 |
| `overPotentialFoam` | 퍼텐셜 유동 해석. overset 격자 |
| `scalarTransportFoam` | 수동적 스칼라량의 수송 방정식 해석 |

![다이어그램 - 기본 해석 솔버](./figures/0_diagram_basic.png)

### 3.2 비압축성 유동 해석

| 솔버 | 설명 |
| :--- | :--- |
| `icoFoam` | 비정상상태 비압축성 층류 유동 해석. 뉴턴 유체 |
| `nonNewtonianIcoFoam` | 비정상상태 비압축성 층류 유동 해석. 비뉴턴 유체 |
| `simpleFoam` | 정상상태 비압축성 난류 유동 해석 |
| `overSimpleFoam` | 정상상태 비압축성 난류 유동 해석. 난류 모델링. overset 격자 |
| `porousSimpleFoam` | 정상상태 비압축성 난류 유동 해석. 다공성 매질 모델링. 다중/이동 참조계(MRF) |
| `SRFSimpleFoam` | 정상상태 비압축성 난류 유동 해석. 비뉴턴 유체. 단일 회전 참조계(SRF) |
| `pisoFoam` | 비정상상태 비압축성 난류 유동 해석. PISO 알고리즘 활용 |
| `pimpleFoam` | 비정상상태 비압축성 난류 유동 해석. PISO+SIMPLE 알고리즘 사용. 뉴턴 유체. 이동 격자 |
| `overPimpleDyMFoam` | 비정상상태 비압축성 유동 해석. PISO+SIMPLE 알고리즘 사용. 뉴턴 유체. 이동 격자. overset 격자 |
| `SRFPimpleFoam` | 비정상상태 비압축성 유동 해석. PISO+SIMPLE 알고리즘 사용. 단일 회전 참조계. 큰 시간 간격 |
| `shallowWaterFoam` | 비정상상태 비점성 얕은 물 방정식 해석. 회전 운동 |
| `boundaryFoam` | 정상상태 비압축성 1차원 난류 유동 해석. 입구에서의 경계층 조건을 생성하기 위해 활용 |
| `adjointShapeOptimizationFoam` | 정상상태 비압축성 난류 유동. 비뉴턴 유체. adjoint 식을 이용해 예측한 압력 강하 유발 영역에 blockage를 적용하여 유로 형상 최적화 |

![다이어그램 - 비압축성 유동 해석 솔버](./figures/0_diagram_incompressible.png)

### 3.3 압축성 유동 해석

| 솔버 | 설명 |
| :--- | :--- |
| `rhoSimpleFoam` | 정상상태 압축성 난류 유동 |
| `overRhoSimpleFoam` | 정상상태 압축성 난류 유동. overset 격자 |
| `rhoPorousSimpleFoam` | 정상상태 압축성 난류 유동. 다공성 매질 모델링 |
| `rhoPimpleFoam` | 비정상상태 압축성 난류 유동. 이동 격자 및 격자 위상 변환 |
| `overRhoPimpleDyMFoam` | 비정상상태 압축성 층류/난류 유동. overset 격자 |
| `rhoCentralFoam` | 밀도 기반 압축성 유동 해석. 중간-풍상 도법 사용. 이동 격자 및 격자 위상 변환 |
| `sonicFoam` | 비정상상태 압축성 기체의 천음속/초음속 난류 유동 |
| `sonicLiquidFoam` | 비정상상태 압축성 액체의 천음속/초음속 난류 유동 |
| `sonicDyMFoam` | 비정상상태 압축성 기체의 천음속/초음속 난류 유동. 이동 격자 및 격자 위상 변환 |

![다이어그램 - 압축성 유동 해석 솔버](./figures/0_diagram_compressible.png)

### 3.4 열전달 해석

| 솔버 | 설명 |
| :--- | :--- |
| `buoyantSimpleFoam` | 정상상태 압축성 부력 난류 유동 해석 |
| `buoyantBoussinesqSimpleFoam` | 정상상태 비압축성 부력 난류 유동 해석 |
| `buoyantPimpleFoam` | 비정상상태 압축성 부력 난류 유동 해석. 이동 격자 및 격자 위상 변환 |
| `buoyantBoussinesqPimpleFoam` | 비정상상태 비압축성 부력 난류 유동 해석. 이동 격자 및 격자 위상 변환 |
| `overBuoyantPimpleDyMFoam` | 비정상상태 압축성 부력 난류 유동 해석. overset 격자 |
| `chtMultiRegionFoam` | 비정상상태 복합 열전달 해석(부력 난류 유동 + 고체 열전도) |
| `chtMultiRegionSimpleFoam` | 정상상태 복합 열전달 해석(부력 난류 유동 + 고체 열전도) |
| `chtMultiRegionTwoPhaseEulerFoam` | 비정상상태 복합 열전달 해석(부력 난류 유동 + 고체 열전도) |
| `solidFoam` | 고체 내 열전달 및 열역학 해석 |
| `thermoFoam` | 고정된(frozen) 유동장 내 열전달 및 열역학 해석 |

![다이어그램 - 열전달 해석 솔버](./figures/0_diagram_heat.png)

### 3.5 연소 해석

| 솔버 | 설명 |
| :--- | :--- |
| `chemFoam` | 화학 반응 문제 해석. 단일 격자 셀을 대상으로 초기 조건으로부터 생성된 장 변수를 사용하여, 다른 화학 반응 솔버와 비교하는 목적으로 설계됨 |
| `coldEngineFoam` | 내연 기관 내 냉간 유동 해석 |
| `fireFoam` | 화재, 난류 확산 화염(고체 연료 반응, 표면 막, 열분해 모델링 포함) 해석 |
| `PDRFoam` | 압축성 난류 예혼합/부분예혼합 연소 해석 |
| `reactingFoam` | 화학 반응을 이용한 연소 해석 |
| `rhoReactingFoam` | 화학 반응을 이용한 연소 해석. 밀도 기반 열역학 패키지 활용 |
| `rhoReactingBuoyantFoam` | 화학 반응을 이용한 연소 해석. 밀도 기반 열역학 패키지 활용. 부력 처리 기법 강화 |
| `XiFoam` | 압축성 난류 예혼합/부분예혼합 연소 해석 |
| `XiDyMFoam` | 압축성 난류 예혼합/부분예혼합 연소 해석. 이동 격자 |

![다이어그램 - 연소 해석 솔버](./figures/0_diagram_combustion.png)

### 3.6 다상 유동 해석

#### 3.6.1 VOF 모델

| 솔버 | 설명 |
| :--- | :--- |
| `interFoam` | VOF 다상유동 해석. 2유체. 비압축성 등온 유체. 이동 격자 및 격자 위상 변환. 적응형 격자 재생성 |
| `overInterDyMFoam` | `interFoam` + overset 격자 |
| `interMixingFoam` | `interFoam` + 3유체(중 2개는 서로 투과할 수 있음) |
| `interIsoFoam` | `interFoam` + isoAdvector를 이용한 계면 추적 성능 강화 |
| `interPhaseChangeFoam` | `interFoam` + 상변화(cavitation 등) |
| `interPhaseChangeDyMFoam` | `interPhaseChangeFoam` + 이동 격자 및 격자 위상 변환. 적응형 격자 재생성 |
| `overInterPhaseChangeDyMFoam` | `interPhaseChangeDyMFoam` + overset 격자 |
| `interCondensatingEvaporatingFoam` | VOF 다상유동 해석. 2유체. 비압축성 비등온 유체. 응축, 증발 등의 상변화 해석 |
| `compressibleInterFoam` | VOF 다상유동 해석. 2유체. 압축성 비등온 유체 |
| `overCompressibleInterDyMFoam` | `compressibleInterFoam` + overset 격자 |
| `multiphaseInterFoam` | VOF 다상유동 해석. N개 유체. 비압축성 유체 |
| `compressibleMultiphaseInterFoam` | VOF 다상유동 해석. N개 유체. 압축성 비등온 유체 |
| `twoLiquidMixingFoam` | VOF 다상유동 해석. 두 혼합성 액체 |
| `icoReactingMultiphaseInterFoam` | VOF 다상유동 해석. N개 유체. 화학 반응 |

#### 3.6.2 Euler-Euler 모델

| 솔버 | 설명 |
| :--- | :--- |
| `twoPhaseEulerFoam` | 오일러-오일러 다상유동 해석. 2유체. 압축성. 1개 유체는 분산상 |
| `reactingTwoPhaseEulerFoam` | `twoPhaseEulerFoam` + 화학 반응 |
| `multiphaseEulerFoam` | `twoPhaseEulerFoam` + 3유체 이상으로 확장 |

#### 3.6.3 Mixture 모델

| 솔버 | 설명 |
| :--- | :--- |
| `driftFluxFoam` | Mixture 모델. 2유체. 비압축성 유체. 상 간 상대속도를 drift-flux |

#### 3.6.4 캐비테이션 모델

| 솔버 | 설명 |
| :--- | :--- |
| `cavitatingFoam` | 비정상상태 캐비테이션 해석. 균질 평형 혼합 모델. 압축성 액체 |
| `cavitatingDyMFoam` | `cavitatingFoam` + 이동 격자 |

#### 3.6.5 자유 표면 모델

| 솔버 | 설명 |
| :--- | :--- |
| `potentialFreeSurfaceFoam` | 자유 표면 유동 해석. 퍼텐셜 유동 기반 |
| `potentialFreeSurfaceDyMFoam` | `potentialFreeSurfaceFoam` + 이동 격자 |

![다이어그램 - 다상 유동 해석 솔버](./figures/0_diagram_multiphase.png)

### 3.7 입자 거동 해석

| 솔버 | 설명 |
| :--- | :--- |
| `kinematicParcelFoam` | 비반응성 운동학적 입자 추적. 압축성/비압축성 유동 |
| `icoUncoupledKinematicParcelFoam` | 비정상상태 비압축성 유동 + 비결합 운동학적 입자 |
| `icoUncoupledKinematicParcelDyMFoam` | `icoUncoupledKinematicParcelFoam` + 이동 격자 |
| `uncoupledKinematicParcelFoam` | 압축성 유동 + 비결합 운동학적 입자 |
| `reactingParcelFoam` | 반응성 입자 해석. 증발 및 표면 반응 |
| `reactingHeterogenousParcelFoam` | 비균질 반응성 입자 해석 |
| `sprayFoam` | 분무(스프레이) 해석. 액적 추적 및 증발 |
| `sprayDyMFoam` | `sprayFoam` + 이동 격자 |
| `simpleSprayFoam` | 단순화된 분무 해석 |
| `coalChemistryFoam` | 석탄 연소 해석. 휘발 및 표면 반응 |
| `simpleCoalParcelFoam` | 단순화된 석탄 입자 해석 |
| `DPMFoam` | Dense Particulate Model. 고밀도 입자 유동 |
| `MPPICFoam` | Multi-Phase Particle-In-Cell 방법. 다상 PIC |

![다이어그램 - 입자 거동 해석 솔버](./figures/0_diagram_lagrangian.png)

### 3.8 표면/막 해석

| 솔버 | 설명 |
| :--- | :--- |
| `liquidFilmFoam` | 표면 위 액체 박막 유동 해석 |
| `surfactantFoam` | 계면 활성제(surfactant) 수송 해석 |
| `sphereSurfactantFoam` | 구 표면 위 계면 활성제 수송 해석 |

![다이어그램 - 표면/막 해석 솔버](./figures/0_diagram_finite.png)

### 3.9 전자기장 해석

| 솔버 | 설명 |
| :--- | :--- |
| `electrostaticFoam` | 정전기장 및 전하장 해석 |
| `magneticFoam` | 영구 자석으로부터 생성되는 자기장 해석 |
| `mhdFoam` | 자기수력학 해석. 비압축성 층류 유동 |

![다이어그램 - 전자기장 해석 솔버](./figures/0_diagram_electro.png)

### 3.10 분자 동역학 해석

| 솔버 | 설명 |
| :--- | :--- |
| `dsmcFoam` | direct simulation of Monte Carlo solver |
| `mdFoam` | 유체역학을 위한 분자 동역학 해석 |
| `mdEquilibrationFoam` | 분자 동역학 계의 전처리 |

![다이어그램 - 분자 동역학 해석 솔버](./figures/0_diagram_discrete.png)

### 3.11 직접 수치 모사

| 솔버 | 설명 |
| :--- | :--- |
| `dnsFoam` | 등방성 난류의 직접 수치 모사 |

![다이어그램 - DNS 해석 솔버](./figures/0_diagram_direct.png)

### 3.12 응력-변형률 해석

| 솔버 | 설명 |
| :--- | :--- |
| `solidDisplacementFoam` | 비정상상태 고체 변위(displacement) 해석. 선형 탄성 및 열탄성 |
| `solidEquilibriumDisplacementFoam` | 정상상태 고체 평형 변위 해석. 선형 탄성 |

![다이어그램 - 구조 해석 솔버](./figures/0_diagram_stress.png)

### 3.13 금융 해석

| 솔버 | 설명 |
| :--- | :--- |
| `financialFoam` | Black-Scholes 방정식 기반 옵션 가격 결정 |

![다이어그램 - 금융 해석 솔버](./figures/0_diagram_financial.png)
