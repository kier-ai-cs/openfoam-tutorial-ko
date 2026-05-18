# 예제 2. 내장 격자 생성 도구 (`blockMesh`)

**참고**:

- 이 책은 Technische Universität Wien, Institute of Chemical, Environmental & Bioscience Engineering의 **OpenFOAM Basic Training, 5th edition (2019)** ([www.cfd.at/tutorials](https://www.cfd.at/tutorials))를 번역한 것이다.
- 필요한 경우에는 글을 의역하거나 내용을 보충하였으나, 전체적인 흐름을 바꾸는 정도의 수정은 없었으므로 수정 내용을 별도로 표시하지는 않았다.
- 이 자료는 [CC BY-NC-SA 3.0](https://creativecommons.org/licenses/by-nc-sa/3.0/) 라이선스 하에 배포되며, 원저작권은 TU Wien에 있다. 기여자 및 라이선스 상세 내용은 [LICENSE.md](./LICENSE.md)를 참고한다.
- 이 자료는 ESI Group, ESI-OpenCFD 또는 OpenFOAM Foundation에 의해 승인된 자료가 아니다.

**호환성**:

- 원서에 수록된 명령어 및 소스 코드는 OpenFOAM 7을 기반으로 작성되었다.
- 원서에서는 OpenFOAM v1906에 대해서도 호환성을 확인하였다. 명령어 및 소스 코드를 변경할 필요가 있는 경우에는 Note에 별도로 설명하였다.
- 번역본은 OpenFOAM v2412를 기반으로 호환성을 확인하였다. 명령어 및 소스 코드를 변경할 필요가 있는 경우에는 Note에 별도로 설명하였다.

## 목차

1. 배경
   - 1.1 격자(mesh)
   - 1.2 유한체적법(finite volume method)
   - 1.3 수송 방정식의 이산화
   - 1.4 `rhoPimpleFoam`
2. 예제 개요
3. 해석
   - 3.1 전처리
   - 3.2 시뮬레이션 실행
   - 3.3 후처리

## 1. 배경

### 1.1 격자(mesh)

유체 유동 및 열전달 현상을 기술하는 질량, 에너지, 운동량 보존 방정식은 편미분 방정식이다. 이 편미분 방정식은 매우 단순화된 경우를 제외하면 대개 해석적으로 풀리지 않는다. 이러한 문제를 풀기 위해 이산화(discretization) 기법이 도입된다. 유동 영역을 작은 하위 영역들로 분할하고, 각각의 하위 영역에서 지배 방정식을 근사적으로 계산한다. 지배 방정식을 계산하기 위한 방법 중 하나가 이 책에서 다루는 유한체적법(finite volume method)이다. 분할된 각각의 하위 영역들을 격자 셀(grid cell)이라고 하고, 이 격자 셀들의 집합이 격자(mesh)를 구성한다.

### 1.2 유한체적법(finite volume method)

[예제 1](./01_기초-케이스-설정.md)에 기술한 것처럼, OpenFOAM은 유한체적법에 기반한 코드이다. 유한체적법은 보존 물리량(conserved fluid property) $\varphi$ (= $u$, $T$, $p$, ...)의 수송 방정식으로부터 시작한다. 수송 방정식의 일반적인 형태는 아래와 같다.

$$\frac{\partial \left(\rho\varphi\right)}{\partial t} + \nabla \cdot \left(\rho\varphi\vec{u}\right) = \nabla \cdot \left(\Gamma \nabla\varphi\right) + S_\varphi$$

좌변의 두 항은 각각 유한체적 내부에서의 $\varphi$의 시간 변화율과 외부 장(field)에 의해 유한체적 외부로 흘러나가는 $\varphi$의 전달률을 의미하고, 우변의 두 항은 각각 확산에 의해 유한체적 내부로 유입되는 $\varphi$의 전달률과 유한체적 내부에서 발생하는 $\varphi$의 생성률을 의미한다. 즉, 수송 방정식은 문제의 영역 내부에서 물리량의 보존 법칙을 표현하는 식이다.

3차원 제어체적(control volume, CV)에서 수송 방정식을 적분하는 것이 유한체적법의 중요한 특징이다. 격자 셀로 분할된 형상에서 각 물리량은 이산화된 지점에서 계산된다. 격자점(node)로 둘러싸인 작은 체적을 격자 셀이라 한다.

각 격자 셀에서, 발산(divergence) 항의 체적 적분(volume integral)은 가우스 발산 정리에 의해 면 적분(surface integral)로 대체되어, 각 면에서의 선속(flux)으로 계산된다. 이는 각 격자 셀에 유출·입하는 선속에 대한 보존 법칙을 만족하도록 할 뿐만 아니라, 비정렬 격자에서 각 물리량의 수지(balance)를 정식화할 때도 유리하다.

비정상 상태(unsteady or transient state)의 문제를 해결할 때는 시간 항 역시 작은 시간 간격 $\Delta t$ 동안 시간에 대해 적분해야 한다. 시간 항의 적분을 포함하여 수송 방정식의 가장 일반적인 적분 형태는 아래와 같이 나타난다.

$$\int_{\Delta t} \frac{\partial}{\partial t} \left( \int_{CV} \rho\varphi dV \right) dt + \int_{\Delta t}\int_{A} \vec{n}\cdot\left(\rho\varphi\vec{u}\right)dAdt = \int_{\Delta t}\int_A \vec{n}\cdot \left(\Gamma \nabla\varphi \right) dAdt + \int_{\Delta t}\int_{CV} S_\varphi dVdt$$

### 1.3 수송 방정식의 이산화

수송 방정식을 이산화하는 것은 수송 방정식을 디지털 컴퓨터에 기반하여 수치적으로 계산할 때 비용 면에서 가장 효율적이고 빠르기 때문에 유한체적법에서도 가장 중요한 요소이다. 이산화는 격자를 이용해서 이루어진다.

OpenFOAM은 데카르트 좌표계에 기반하여 생성된 단순한 정렬 격자와 곡률 및 형상의 복잡성을 취급할 수 있는 복잡한 비정렬 격자를 모두 사용할 수 있다. 격자는 `blockMesh`나 `snappyHexMesh`와 같이 OpenFOAM에 내장된 격자 생성 도구를 사용하거나, GAMBIT과 같은 외부 소프트웨어를 사용해서 생성한다. 이 예제에서는 OpenFOAM의 내장 격자 생성 도구인 `blockMesh`의 사용법에 대해 설명한다. `snappyHexMesh`의 사용법에 대해서는 예제 12를 참고하라.

이 예제에서는 `icoFoam`보다 약간 더 복잡한 `rhoPimpleFoam`을 사용한다. `rhoPimpleFoam`은 압축성 유동 솔버이다.

### 1.4 `rhoPimpleFoam`

`rhoPimpleFoam`은 압축성 유체의 아음속/초음속 난류 유동을 해석하는 비정상 상태 솔버이다.

> **Note:** OpenFOAM v1906:
> `rhoPimpleFoam` $\rightarrow$ `sonicFoam`

> **Note:** OpenFOAM v2412:
> `rhoPimpleFoam` $\rightarrow$ `sonicFoam`

## 2. 예제 개요

### 예제 구성

`rhoPimpleFoam`을 이용하여 전향 계단(forward facing step)의 유동을 10초 간 해석한다.

> **Note:** OpenFOAM v1906:
> `rhoPimpleFoam` $\rightarrow$ `sonicFoam`

> **Note:** OpenFOAM v2412:
> `rhoPimpleFoam` $\rightarrow$ `sonicFoam`

### 목표

- `blockMesh`의 사용 방법을 이해한다.
- 좌표계를 이용해 점(vertex)을 정의하고, 점을 이용해 면(surface)과 체적(volume)을 정의한다.

### 데이터 처리

- 해석 결과를 ParaView로 가져오고, 격자와 해석 결과를 분석한다.

## 3. 해석

### 3.1 전처리

#### 3.1.1 예제 복사

아래의 예제를 작업 디렉토리로 복사한다.

```bash
$FOAM_TUTORIALS/compressible/rhoPimpleFoam/laminar/forwardstep
```

> **Note:** OpenFOAM v1906:
>
> ```bash
> $FOAM_TUTORIALS/compressible/sonicFoam/laminar/forwardstep
> ```

> **Note:** OpenFOAM v2412:
>
> ```bash
> $FOAM_TUTORIALS/compressible/sonicFoam/laminar/forwardstep
> ```

#### 3.1.2 케이스 구조

**0/ 디렉토리**

T 파일은 온도의 초기값을 정의한다. 압력과 온도의 내부장은 1로 설정하고, 유속의 초기 조건과 입구에서의 경계 조건은 3으로 설정한다.

> **Note:** `rhoPimpleFoam`은 압축성 유동 솔버이기 때문에, p의 단위로 환산 압력이 아니라 실제 압력의 단위($\mathrm{kg}$ $\mathrm{m^{-1}}$ $\mathrm{s^{-2}}$)를 사용한다.

> **Note:** 이 예제는 오직 수치해석의 연습을 위한 예제임을 주의해야 한다. 계산된 압력 값을 주의 깊게 관찰하라.

**constant/ 디렉토리**

thermophysicalProperties 파일은 압축성 기체의 물성치를 정의한다.

```cpp
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * *//
thermoType
{
    type            hePsiThermo;
    mixture         pureMixture;
    transport       const;
    thermo          hConst;
    equationOfState perfectGas;
    specie          specie;
    energy          sensibleInternalEnergy;
}
// Note: these are the properties for a "normalized" inviscid
//      gas for which the speed of sound is 1 m/s at a
//      temperature of 1K and gamma = 7/5
mixture
{
    specie
    {
        molWeight   11640.3;
    }
    thermodynamics
    {
        Cp          2.5;
        Hf          0;
    }
    transport
    {
        mu          0;
        Pr          1;
    }
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * *//
```

`thermoType`은 기체의 열물리적 특성을 계산하기 위한 서브모델을 설정한다.

- `type`: 내재된 열물리적 모델을 설정한다.

- `mixture`: 혼합물의 종류를 정의한다. 혼합물의 종류에는 순수 혼합물(pure mixture), 균질 혼합물(homogeneous mixture), 화학 반응 혼합물(reacting mixture) 등이 있다.

- `transport`: 수송 모델을 정의한다. 이 예제에서는 상수 값을 사용한다.

- `thermo`: 열용량을 계산하기 위한 모델을 설정한다. 이 예제에서는 상수 값을 사용한다.

- `equationOfState`: 기체의 압축성을 기술하는 모델을 설정한다. 이 예제에서는 `perfectGas`를 선택하여, 기체를 이상 기체로 간주한다.

- `energy`: 솔버가 에너지 방정식으로 엔탈피 방정식과 내부에너지 방정식 중 어떤 것을 계산할지 설정한다.

기체의 각 열물리적 특성에 적용할 모델을 정의한 후, `mixture` 하위 딕셔너리에서 각 모델이 사용할 상수와 계수를 정의한다. 예를 들어, `molWeight`는 기체의 몰질량을 의미하고, `Cp`는 열용량, `Hf`는 융해열(heat of fusion), `mu`는 점성(dynamic viscosity), `Pr`은 Prandtl수를 나타낸다.

turbulenceProperties 파일은 난류 모델을 정의한다. 이 예제는 층류 유동을 해석한다.

```cpp
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * *//
simulationType      laminar;
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * *//
```

**system/ 디렉토리**

이 예제에서는 외부 소프트웨어로 생성된 격자를 불러들이지 않고, OpenFOAM의 내장 격자 생성 도구인 `blockMesh`로 격자를 생성한다. `blockMesh`는 system/ 디렉토리에 있는 blockMeshDict 파일로부터 형상과 격자 정보를 불러들인다.

```bash
nano blockMeshDict
```

```cpp
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * *//
convertToMeters     1;
vertices
(
    (0 0 -0.05)     // 0
    (0.6 0 -0.05)   // 1
    (0 0.2 -0.05)   // 2
    (0.6 0.2 -0.05) // 3
    (3 0.2 -0.05)   // 4
    (0 1 -0.05)     // 5
    (0.6 1 -0.05)   // 6
    (3 1 -0.05)     // 7
    (0 0 0.05)      // 8
    (0.6 0 0.05)    // 9
    (0 0.2 0.05)    // 10
    (0.6 0.2 0.05)  // 11
    (3 0.2 0.05)    // 12
    (0 1 0.05)      // 13
    (0.6 1 0.05)    // 14
    (3 1 0.05)      // 15
);
blocks
(
    hex (0 1 3 2 8 9 11 10) (25 10 1) simpleGrading (1 1 1)
    hex (2 3 6 5 10 11 14 13) (25 40 1) simpleGrading (1 1 1)
    hex (3 4 7 6 11 12 15 14) (100 40 1) simpleGrading (1 1 1)
);
edges
(
);
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * *//
```

OpenFOAM은 SI 단위계를 사용한다. 점의 좌표가 SI 단위로 작성되지 않았다면 `convertToMeters`를 이용하여 단위를 변환할 수 있다. `convertToMeters`의 값은 각 좌표값을 미터 단위로 바꾸기 위해 곱해주어야 하는 상수를 의미한다. 예를 들어,

```cpp
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * *//
convertToMeters     0.001;
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * *//
```

위의 값은 점이 밀리미터 단위로 정의되어 있으며, 이를 미터 단위로 변환하기 위해 0.001을 곱해준다는 의미이다.

`vertices`는 형상을 구성하는 꼭지점의 좌표를 정의하며, 각 꼭지점은 0번부터 순서대로 번호를 매겨 저장된다. 예를 들어, 위 예제에서 점 (0 0 -0.05)는 0번 점이고, 점 (0.6 1 -0.05)는 6번 점이다.

`blocks`은 블록을 정의한다. 각 블록의 앞에 있는 숫자의 배열은 각 블록을 구성하는 꼭지점의 번호를 나타낸다. 예를 들어, 첫 번째 블록은 `(0 1 3 2 8 9 11 10)`번 점들로 구성된다.

블록을 정의한 후, 격자는 모든 방향으로 각각 정의된다. 예를 들어, `(25 10 1)`은 해당 블록이 x 방향으로 25개, y 방향으로 10개, z 방향으로 1개의 격자를 갖도록 분할된다는 의미이다. OpenFOAM에서는 2차원 해석을 수행하더라도 격자는 반드시 3차원으로 생성해야 한다. 단, 해석을 수행하지 않는 방향으로는 격자를 1개만 생성한다. 예를 들어, 이 예제는 x-y 평면 상에서 2차원 해석을 수행하기 때문에, z 방향으로는 격자를 1개만 생성한다.

마지막 부분인 `simpleGrading (1 1 1)`은 사이즈 함수를 나타낸다.

```cpp
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * *//
boundary
(
    inlet
    {
        type patch;
        faces
        (
            (0 8 10 2)
            (2 10 13 5)
        );
    }
    outlet
    {
        type patch;
        faces
        (
            (4 7 15 12)
        );
    }
    bottom
    {
        type symmetryPlane;
        faces
        (
            (0 1 9 8)
        );
    }
    top
    {
        type symmetryPlane;
        faces
        (
            (5 13 14 6)
            (6 14 15 7)
        );
    }
    obstacle
    {
        type patch;
        faces
        (
            (1 3 11 9)
            (3 4 12 11)
        );
    }
);
mergePatchPairs
(
);
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * *//
```

`boundary`는 경계면을 정의한다. 각 경계면을 구성하는 꼭지점과 경계면의 종류 및 이름을 정의한다.

> **Note:** 꼭지점을 이어서 면을 생성할 때는, 형상의 내부에서 그 면을 바라볼 때 시계 방향 순서대로 꼭지점을 선정해야 한다.

### 3.2 시뮬레이션 실행

시뮬레이션을 실행하기 전에 격자가 생성되어야 한다. 이전 단계에서 형상 및 격자 생성 정보가 설정되었다. 설정된 정보를 바탕으로 형상 및 격자를 생성하기 위해, 아래의 명령어를 케이스의 메인 디렉토리에서 실행한다.

```bash
blockMesh
```

생성된 격자는 constant/polyMesh/ 폴더에 저장된다. 시뮬레이션을 실행하기 위해, 케이스 메인 디렉토리에서 솔버의 이름을 입력하고 실행한다.

```bash
rhoPimpleFoam
```

> **Note:** OpenFOAM v1906:
>
> ```bash
> sonicFoam
> ```

> **Note:** OpenFOAM v2412:
>
> ```bash
> sonicFoam
> ```

### 3.3 후처리

ParaView를 통해, 3개의 블록으로 생성된 격자를 확인할 수 있다.

![`blockMesh`로 생성한 격자](./figures/2_mesh.jpg)

> **Note:** ParaView에서 `Slice`를 생성하면, 사각 격자를 생성했음에도 그 평면에서의 격자는 삼각 격자로 나타난다. 이는 ParaView가 가시화를 위해 사각 격자를 2개의 삼각 격자로 표현하기 때문에 발생하는 현상이다. `Slice`를 생성할 때  `Properties > Triangulate the slice`를 해제하면 위 그림처럼 원본 격자를 나타낼 수 있다.

해석 결과는 아래와 같다.

![시간 별 압력, 속도, 온도의 등치선](./figures/2_result.jpg)
