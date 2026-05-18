# 예제 8. 다상 유동 (Volume of fluid, VOF)

**참고**:

- 이 책은 Technische Universität Wien, Institute of Chemical, Environmental & Bioscience Engineering의 **OpenFOAM Basic Training, 5th edition (2019)** ([www.cfd.at/tutorials](https://www.cfd.at/tutorials))를 번역한 것이다.
- 필요한 경우에는 글을 의역하거나 내용을 보충하였으나, 전체적인 흐름을 바꾸는 정도의 수정은 없었으므로 수정 내용을 별도로 표시하지는 않았다.
- 이 자료는 [CC BY-NC-SA 3.0](https://creativecommons.org/licenses/by-nc-sa/3.0/) 라이선스 하에 배포되며, 원저작권은 TU Wien에 있다. 기여자 및 라이선스 상세 내용은 [LICENSE.md](./LICENSE.md)를 참고한다.
- 이 자료는 ESI Group, ESI-OpenCFD 또는 OpenFOAM Foundation에 의해 승인된 자료가 아니다.

**호환성**:

- 원서에 수록된 명령어 및 소스 코드는 OpenFOAM 7을 기반으로 작성되었다.
- 원서에서는 OpenFOAM v1906에 대해서도 호환성을 확인하였다. 명령어 및 소스 코드를 변경할 필요가 있는 경우에는 Note에 별도로 설명하였다.
- 번역본은 OpenFOAM v2412를 기반으로 호환성을 확인하였다. 명령어 및 소스 코드를 변경할 필요가 있는 경우에는 Note에 별도로 설명하였다.

---

## 목차

1. 배경
   - 1.1 다상 유동
   - 1.2 모델링 접근법
   - 1.3 `interFoam`
2. 예제 개요
3. 해석
   - 3.1 전처리
   - 3.2 시뮬레이션 실행
   - 3.3 후처리

## 1. 배경

이 예제에서는 `interFoam`을 이용해 댐 붕괴(dam break) 문제를 해석한다. 이 문제의 주요 특징은 물과 공기가 분명한 계면(interface)을 통해 분리된다는 점이다. 예제를 시작하기에 앞서, 다상 유동의 기초적인 특징을 다룬다.

### 1.1 다상 유동

다상 유동은 둘 이상의 서로 다른 상(phase)으로 이루어진 유체의 유동이다. 각 상 별로 여러 성분(component)이 있을 수 있다. 다상 유동의 종류는 각 상에 따라 기체-액체, 기체-고체, 액체-고체, 액체-액체, 3상 유동 등 여러 가지가 있다.

다상 유동의 종류는 유동의 시각적 형태에 따라서도 구분할 수 있다. 분산 유동(dispersed flow)은 입자나 액적 등의 분산상(dispersed phase)이 물, 공기 등의 연속상(continuous phase)에 분포하여 움직이는 유동을 의미하고, 박리 유동(separated flow)은 둘 이상의 유체가 모두 연속상으로 존재하면서 서로 혼합되지 않아서(immiscible) 계면(interface)으로 그 경계가 분리되어 있는 유동이다. 혼합 유동(mixed flow)은 반연속적(semi-continuous) 계면과 분산상의 입자가 공존하는 경우이다.

그래서, 다상 유동은 왜 중요한가? 이는 다양한 산업 분야에서 기포탑(bubble column), 흡착(adsorption), 흡수(absorption), 스트리핑탑(stripping column) 등 다상 유동을 활용하기 때문이다. 다상 유동 모델링을 통해 두 상 사이의 접촉면을 극대화하여 공정의 효율을 높이는 작업을 보조할 수 있다.

### 1.2 모델링 접근법

유동 양식(flow regime)이 전환될 수 있기 때문에, 다상 유동의 모델링은 극히 복잡한 작업이다. 문제를 단순화하기 위해 여러 다른 방식의 접근법이 개발되었는데, 이는 크게 라그랑지안(Lagrangian)과 오일러리안(Eulerian) 접근법으로 구분된다. 라그랑지안 접근법은 개별 점 입자의 움직임을 추적하기 때문에, 분산상이 존재하는 경우에 적절하다. 오일러리안 접근법은 주어진 제어 체적 내부에서 유체의 움직임을 관찰하는 방식이다.

아래에서 일반적인 다상 유동 모델링 방법의 일부를 소개한다.

#### 1.2.1 오일러리안-오일러리안 접근법(다유체(multi-fluid) 모델)

오일러리안-오일러리안(Eulerian-Eulerian, EE) 방식에서는 모든 상을 연속체로 취급한다. 따라서 각 상이 연속체로 움직이는 박리 유동의 해석에 적합하다. 각 상은 항력과 양력을 주고받거나 열$\cdot$물질 전달을 통해 상호작용한다. 분산상이 존재하는 경우에도, 각 개별 입자의 궤적을 계산하기보다 입자의 전반적인 움직임을 관찰하고자 한다면 EE 방식을 사용할 수 있다.

EE 방식에서는 각 상의 체적 분율(volume fraction)이라는 개념을 사용한다. 이 체적 분율은 시공간에 대해 연속 함수이며, 그 합은 1이다. 질량, 운동량, 에너지 보존 방정식은 각각의 상에 대해 개별적으로 계산되고, 체적 분율에 대한 수송 방정식이 추가적으로 계산된다. 각 상 간의 연동은 압력과 상간 교환 계수에 의해 이루어진다.

#### 1.2.2 와류 상호작용 모델

와류 상호작용 모델(eddy interaction model)에서, 각 입자는 일련의 와류와 상호작용한다. 입자의 유체 운동은 입자의 궤적을 추적하는 라그랑지안 접근법을 따르고, 다음의 3개 파라미터로 특징지어진다. 1\) 와류 속도(eddy velocity), 2\) 와류 수명(eddy lifetime), 3\) 와류 길이(eddy length)

와류의 수명 $t_e$와 길이 스케일 $l_e$는 국소적인 난류 특성으로부터 추정한다. 길이 스케일과 입자의 속도로부터, 입자가 와류를 지나가는데 걸리는 시간인 와류 주행 시간(eddy transit time) $t_c$를 계산할 수 있다. 그러면, 와류의 수명과 와류 주행 시간 중 작은 것이 입자가 와류와 상호작용하는 시간이라고 할 수 있다.

$$t_{int} = \min(t_e, t_c)$$

상호작용이 일어나는 동안 유체의 요동 속도는 일정하게 유지되며, 그 동안 분산상의 입자는 운동 방정식에 따라 움직인다. 이후 새로운 유체 요동 속도를 업데이트하고, 위 과정이 반복된다.

#### 1.2.3 Volume of Fluid (VOF) 모델

VOF는 오일러리안 다상 유동 모델의 일종으로, 체적 분율 함수(fraction function) $C$을 도입하여 다상 유동을 해석한다. 체적 분율 함수는 해당 상이 각 제어체적에 존재하는가의 여부를 가리킨다. $C=1$인 제어 체적은 전부 해당 상으로 채워져 있다는 의미이고, $C=0$인 제어 체적은 전부 다른 상으로 채워져 있다는 의미이다. $C$의 값이 0과 1 사이이면 그 제어 체적 내부에 두 상의 계면이 존재한다는 의미이다. 따라서, VOF 모델을 이용해서 다상 유동을 해석하는 경우, 상 간 계면을 나타낼 수 있을 만큼 충분히 조밀한 격자를 사용해야 한다.

이 예제의 다상 유동은 `interFoam` 솔버를 이용해 해석한다. 아래는 `interFoam`에 대한 간략한 설명이다.

### 1.3 `interFoam`

`interFoam`은 VOF 모델에 기반한 솔버로, 2개의 비압축성(incompressible), 등온(isothermal), 불혼화(immiscible) 유체의 다상 유동을 해석하는데 적합하다.

---

## 2. 예제 개요

### 예제 구성

- `interFoam`을 이용해 댐 붕괴(dam break) 현상을 2초 간 해석한다.

### 목표

- 2개의 상에 대해 점성, 표면 장력, 밀도를 설정하는 방법을 이해한다.

### 데이터 처리

- 해석 결과를 ParaView로 가져와서 확인한다.

---

## 3. 해석

### 3.1 전처리

#### 3.1.1 예제 복사

아래의 예제를 작업 디렉토리로 복사한다.

```bash
$FOAM_TUTORIALS/multiphase/interFoam/laminar/damBreak/damBreak
```

#### 3.1.2 0/ 디렉토리

0/ 디렉토리에는 alpha.water.orig, p\_rgh, U 파일이 저장되어 있다. alpha.water.orig와 p\_rgh 파일은 압력과 물의 초기 및 경계조건을 설정한다. alpha.water.orig 파일의 사본을 생성하고 이름을 alpha.water로 변경한다. \*.orig 파일은 백업 파일로, OpenFOAM의 솔버는 \*.orig 파일을 사용하지 않는다.

> **Note:** OpenFOAM v1906: alpha.water 파일도 저장되어 있다.

> **Note:** OpenFOAM v2412: 0.orig/ 디렉토리에 alpha.water, p\_rgh, U 파일이 저장되어 있다. 0.orig/ 폴더의 사본을 생성하고 폴더의 이름을 0로 변경하여 사용한다.

아래는 alpha.water 파일의 내용이다.

```cpp
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * *//
dimensions              [0 0 0 0 0 0 0];

internalField           uniform 0;
boundaryField
{
    leftWall
    {
        type            zeroGradient;
    }
    rightWall
    {
        type            zeroGradient;
    }
    lowerWall
    {
        type            zeroGradient;
    }
    atmosphere
    {
        type            inletOutlet;
        inletValue      uniform 0;
        value           uniform 0;
    }
    defaultFace
    {
        type            empty;
    }
```

> **Note:** `inletOutlet`과 `outletInlet`은 유동의 방향을 알 수 없을 때 사용하는 경계 조건으로, 2개의 서로 다른 경계 조건의 조합으로 파생된 경계 조건이다.
>
> - `inletOutlet`: 선속의 방향이 계산 영역의 바깥을 향할 때는 `zeroGradient` 조건으로 작용하고, 선속의 방향이 계산 영역 안쪽을 향할 때는 `fixedValue` 조건으로 작용한다.
> - `outletInlet`: 선속의 방향이 계산 영역의 바깥을 향할 때는 `fixedValue` 조건으로 작용하고, 선속의 방향이 계산 영역 안쪽을 향할 때는 `zeroGradien` 조건으로 작용한다.
>
> 예를 들어, 유동의 출구를 `inletOutlet`으로 설정하고 `inletValue`를 `(0 0 0)`으로 설정하면 유출 선속은 `zeroGradient`로 처리하고 유입 선속은 0으로 처리하여 해당 출구에서 역류(backflow)의 발생을 억제할 수 있다. `inletValue` (또는 `outletValue`)는 해당 경계 조건의 `fixedValue`로 사용될 값이고, `value`는 OpenFOAM 솔버에 변수의 자료형을 전달하기 위한 더미 입력값이다. 예를 들어, `value`의 값으로 `(0 0 0)`을 입력하면 OpenFOAM은 그 변수를 벡터로 상정한다. 현재 `value`의 값은 `0`이므로 OpenFOAM은 이 변수를 스칼라로 상정한다.

#### 3.1.3 constant/ 디렉토리

transportProperties 파일의 하위 딕셔너리를 이용해 각 상의 물성을 설정한다.

```cpp
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * *//
phases (water air);
water
{
    transportModel  Newtonian;
    nu              1e-06;
    rho             1000;
}
air
{
    transportModel  Newtonian;
    nu              1.48e-05;
    rho             1;
}
sigma               0.07;
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * *//
```

각 상 별로 점성을 기술하는 모델과 해당 모델의 파라미터를 설정한다. 어떤 모델을 사용하느냐에 따라 그에 해당하는 하위 딕셔너리에서 모델 파라미터를 읽어들인다. 이 예제에서는 `Newtonian` 모델을 사용하기 때문에 필요한 모델 파라미터는 `nu`이다.

`sigma`는 두 상 사이에 작용하는 표면 장력이다. 이 예제에서는 물과 공기 간의 표면 장력을 사용한다.

g 파일은 중력장의 크기와 방향을 정의한다. 이 예제에서는 -y 방향으로 9.81 $\mathrm{m/s^2}$이다.

```cpp
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * *//
dimensions      [0 1 -2 0 0 0 0];
value           (0 -9.81 0);
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * *//
```

### 3.2 시뮬레이션 실행

```bash
blockMesh
setFields
interFoam
```

### 3.3 후처리

해석 결과는 아래와 같다. (아래 결과는 원래의 격자를 2배 세분할하여 해석한 결과이다.)

![시간 단계 별 물의 체적 분율의 등치선 플롯](./figures/8_result1.jpg)
