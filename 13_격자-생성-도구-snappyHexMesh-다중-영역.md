# 예제 13. 격자 생성 도구 (`snappyHexMesh` - 다중 영역)

**참고**:

- 이 책은 Technische Universität Wien, Institute of Chemical, Environmental & Bioscience Engineering의 **OpenFOAM Basic Training, 5th edition (2019)** ([www.cfd.at/tutorials](https://www.cfd.at/tutorials))를 번역한 것이다.
- 필요한 경우에는 글을 의역하거나 내용을 보충하였으나, 전체적인 흐름을 바꾸는 정도의 수정은 없었으므로 수정 내용을 별도로 표시하지는 않았다.
- 이 자료는 [CC BY-NC-SA 3.0](https://creativecommons.org/licenses/by-nc-sa/3.0/) 라이선스 하에 배포되며, 원저작권은 TU Wien에 있다. 기여자 및 라이선스 상세 내용은 [LICENSE.md](./LICENSE.md)를 참고한다.
- 이 자료는 ESI Group, ESI-OpenCFD 또는 OpenFOAM Foundation에 의해 승인된 자료가 아니다.

**호환성**:

- 원서에 수록된 명령어 및 소스 코드는 OpenFOAM 7을 기반으로 작성되었다.
- 원서에서는 OpenFOAM v1906에 대해서도 호환성을 확인하였다. 명령어 및 소스 코드를 변경할 필요가 있는 경우에는 Note에 별도로 설명하였다.
- 번역본은 OpenFOAM v2412을 기반으로 호환성을 확인하였다. 명령어 및 소스 코드를 변경할 필요가 있는 경우에는 Note에 별도로 설명하였다.

---

## 목차

1. 배경
   - 1.1 다중 영역 모델링
   - 1.2 다중 영역 모델링 기법의 종류
   - 1.3 OpenFOAM에서의 다중 영역 모델링 방법
2. 예제 개요
3. 해석
   - 3.1 전처리
   - 3.2 격자 생성 및 해석 실행
   - 3.3 후처리

## 1. 배경

### 1.1 다중 영역 모델링

다중 영역 모델링은 전체 계산 영역을 여러 개의 개별 영역으로 나누고, 각 영역이 특정 상 또는 물질을 나타내는 것으로 처리하는 기법이다. 다중 영역 모델링 기법은 복합 열전달 해석, 유체-고체 연성 해석, 다상 유동 해석 등에 특히 유용하다. 다중 영역 모델링의 핵심적인 특징은 각 영역마다 서로 다른 수송 방정식을 풀 수 있다는 점으로, 이를 통해 영역 별 물성치의 차이, 물질 별 거동 특성, 그리고 계면에서의 상호작용을 정확하게 표현할 수 있다.

특히 아래와 같은 경우에 다중 영역 모델링이 유용하게 사용될 수 있다.

- 열역학적 특성이 다른 물질들이 공존하는 경우 (열교환기, 전자 장비 냉각, 단열재 등)
- 고체 내 전도 열전달과 유체 내 대류 열전달이 동시에 일어나는 경우 (복합 열전달)
- 여러 개의 상 또는 물질 간의 계면이 존재하는 경우 (고체-액체 계면 등)
- 유체-고체 연성 해석
- 열전달 현상에 의해 영역 별로 다른 지배 방정식을 풀어야 하는 경우

### 1.2 다중 영역 모델링 기법의 종류

다중 영역 문제를 풀기 위해 일반적으로 두 가지 방식이 사용되어 왔다.

- Monolithic solver: 영역 전체를 하나의 연립 행렬 방정식 시스템으로 구성하여 계산. 개별 영역 간 지배 방정식이 서로 강하게 연동되며, 수치적 안정성이 우수한 대신 계산 비용이 높음. 유한요소 해석 등 고정도 해석에 주로 사용.
- Partitioned solver: 각 영역을 별도의 행렬 방정식 시스템으로 분리하여 계산. 개별 영역 간 계면은 각 영역의 경계 조건으로 처리되며, 따라서 계산 비용이 낮음. 각 영역마다 다른 방정식을 풀어야 하지만 계면을 통해 서로 상호작용하는 경우에 사용.

### 1.3 OpenFOAM에서의 다중 영역 모델링 방법

#### **1. 계산 영역 및 격자 영역 정의**

전체 계산 영역을 여러 개의 개별 영역들로 분할한다. 이 영역은 주로 고체와 유체 영역으로 구분된다. 각 영역마다 적절한 물성치를 정의한다. 유체 영역에서는 주로 밀도, 점성 계수, 열전도 계수, 비열 등이며, 고체 영역에서는 주로 밀도, 열전도 계수, 비열 등이다. 격자는 각 영역마다 개별적으로 생성된다.

#### **2. 영역 별 변수 지정**

각 개별 영역마다 해석하는 지배 방정식에 따라 필요한 변수를 정의한다. 각 영역의 열역학적 물성치는 `thermophyscialModels`에서 정의하며, 각 개별 영역 별 초기 조건은 0/ 디렉토리에 정의되어야 한다.

#### **3. 각 개별 영역 별 수송 방정식 해석**

질량, 운동량, 에너지 보존 방정식이 각 개별 영역 별로 계산된다. 일반적으로 유체 영역에서는 Navier-Stokes 방정식, 연속 방정식, 에너지 보존 방정식 등을 계산하며, 고체 영역에서는 열전도 방정식 등을 계산한다. 각 개별 영역마다 수치적 기법을 따로 정의, 관리해야 한다. 시간 간격 크기는 전체 영역에 대해 가장 작은 값을 기준으로 한다.

#### **4. 계면에서의 개별 영역 간 연동 해석**

계면에서의 정보를 각 개별 영역의 경계 조건으로 처리함으로써 영역 간 상호작용을 반영한다. 계면에서는 온도, 열유속, 질량전달률 등이 연속적이어야 한다.

#### **5. 반복 계산을 통한 수렴해 도출**

전체 시스템이 수렴할 때까지 반복 계산을 수행한다. 수치적 안정성을 위해 시간 간격 크기 조절, 솔버 세팅, 완화 계수 도입 등이 필요할 수 있다.

## 2. 예제 개요

### 예제 구성

이 예제에서는 다중 영역으로 구성된 형상을 모델링하고 격자를 생성한 후 이를 이용해 복합 열전달 해석을 수행한다.

### 목표

- `snappyHexMesh`를 이용해 다중 영역에 격자를 생성하는 과정을 이해한다.

### 데이터 처리

- 해석 결과를 ParaView로 확인하고, 플랜지 내부의 열 분포와 주변 유동장을 분석한다.

## 3. 해석

### 3.1 전처리

#### 3.1.1 예제 복사

아래의 예제를 다운로드하여 작업 디렉토리로 복사한다.

```bash
$FOAM_TUTORIALS/heatTransfer/chtMultiRegionFoam/snappyMultiRegionHeater
or
https://github.com/OpenFOAM/OpenFOAM-5.x/tree/master/tutorials
```

constant/ 디렉토리 하위에 triSurface/ 디렉토리를 생성한다.

```bash
mkdir -p $CASE/constant/triSurface
```

아래의 \*.stl 파일을 triSurface/ 디렉토리에 복사한다.

```bash
$FOAM_TUTORIALS/resources/geometry/geom.stl.gz
```

#### 3.1.2 0/ 디렉토리

다중 영역 해석 시, 단일 영역 해석의 경우와 달리 0/ 디렉토리에 각 개별 영역에 대한 하위 디렉토리가 개별적으로 존재하며, 각 개별 영역의 초기 조건 및 경계 조건 파일도 각각의 개별 하위 디렉토리에 저장된다. 개별 영역 별 하위 디렉토리는 격자 생성 및 분할 후 자동으로 생성되며, 그 전에 직접 생성할 수도 있다. 각 개별 영역의 초기 조건 및 경계 조건은 `changeDictionary`를 이용해 수정 및 반영하며, 이에 대한 자세한 설명은 후술한다.

> **Note:** 0/ 디렉토리에는 해석에 실제로 쓰이지 않는 더미 파일도 일부 포함되어 있다.

> **Note:** OpenFOAM v1906: 0.orig/ 폴더의 사본을 생성하고 폴더의 이름을 0로 변경한다.

> **Note:** OpenFOAM v2412: 0.orig/ 폴더의 사본을 생성하고 폴더의 이름을 0로 변경한다.

#### 3.1.3 constant/ 디렉토리

constant/ 디렉토리 또한 각 개별 영역 별로 하위 디렉토리를 가진다. 이 예제의 경우, 전체 계산 영역이 5개의 개별 영역(bottomAir, heater, leftSolid, rightSolid, topAir)으로 분할되어 있다. 각 영역 별 하위 디렉토리에는 각 영역의 물성치(thermophysicalProperties), 난류 특성(turbulenceProperties), 복사 열전달 특성(radiationProperties) 등에 대한 딕셔너리가 저장된다.

constant/ 디렉토리 하위의 polyMesh/ 디렉토리에는 전체 계산 영역에 대한 원본 격자가 저장되고, 각 개별 영역 별 디렉토리 하위의 polyMesh/ 디렉토리에는 각 영역에 해당하는 분할된 격자, 인접 영역과의 새로운 경계 정보가 저장된다.

polyMesh/ 디렉토리와는 달리, triSurface/ 디렉토리는 하나만 존재한다. 격자 생성에 필요한 모든 \*.stl 파일은 이 triSurface/ 디렉토리에 저장된다.

regionProperties 파일에서는 각 영역의 물리적 상(phase)이 명시되어 있다. 이 예제에서는 아래와 같이 bottom air 영역과 top air 영역은 유체 상으로 지정되어 있으며, heater, left solid, right solid 영역은 고체 상으로 지정되어 있다.

```cpp
//**************************************//
regions
(
    fluid (bottomAir topAir)
    solid (heater leftSolid rightSolid)
);
//**************************************//
```

> **Note:** OpenFOAM v1906: constant/topAir/ 디렉토리의 g 파일을 constant/ 디렉토리로 복사하여 사용한다.

#### 3.1.4 system/ 디렉토리

system/ 디렉토리 또한 각 개별 영역 별로 하위 디렉토리를 가지며, 각 영역 별 설정 파일(fvSolution, fvSchemes, decomposeParDict 등)이 각 하위 디렉토리에 저장된다. system/ 디렉토리에 저장된 파일 중 fvSchemes은 실제로는 쓰이지 않는 더미 파일이지만, fvSolution에서는 PIMPLE 알고리즘의 outer corrector의 수 `nOuterCorrectors`를 지정한다. controlDict 파일은 하나만 존재하며, system/ 디렉토리에 저장된다.

> **Note:** 병렬 계산 시, 모든 개별 영역에 대한 decomposeParDict 설정이 메인 system/ 디렉토리에 저장된 decomposeParDict 파일의 설정과 동일해야 한다. 단, 격자를 생성할 때는 메인 system/ 디렉토리의 decomposeParDict 파일만 사용하기 때문에 격자 생성에 대한 설정이 모두 동일할 필요는 없다.

다중 영역에 대해 격자를 생성할 때 필요한 파일은 단일 영역에 대해 필요한 것과 거의 동일하며, snappyHexMeshDict 파일에만 약간 수정이 필요하다.

- `locationInMesh`: 다중 영역 격자 생성 시, `locationInMesh`의 좌표값 자체는 사용되지 않지만, 플레이스홀더로 정의되어 있어야 한다.

- `refinementSurfaces`: 각 개별 영역을 정의한다. 예를 들어, bottom air 영역의 경우, bottomAir.stl 파일 내부의 모든 면과 셀이 `bottomAir` 플래그로 표시된다. 이 표시는 각각 `faceZone`, `cellZone`에 적용된다. 각 영역에 대한 \*.stl 파일은 반드시 닫힌 형상이어야 한다.

```cpp
//**************************************//
castellatedMeshControls
{
    maxLocalCells 100000;
    maxGlobalCells 2000000;
    minRefinementCells 10;
    nCellsBetweenLevels 2;
    features
    (
        {
            file "bottomAir.eMesh";
            level 1;
        }
        …
        {
            file "topAir.eMesh";
            level 1;
        }
    );
    refinementSurfaces
    {
        bottomAir
        {
            level (1 1);
            faceZone bottomAir;
            cellZone bottomAir;
            cellZoneInside inside;
        }
        …
        rightSolid
        {
            level (1 1);
            faceZone rightSolid;
            cellZone rightSolid;
            cellZoneInside inside;
        }
    }
    resolveFeatureAngle 30;
    refinementRegions
    {
    }
    locationInMesh (0.01 0.01 0.01);
    allowFreeStandingZoneFaces false;
}
//**************************************//
```

격자를 생성하고 영역을 분할한 후에는, 0/ 디렉토리 하위의 영역 별 디렉토리에서 각 영역에 대한 초기 조건과 경계 조건을 수동으로 설정할 수 있다. 이 과정은 changeDictionary 기능을 이용해 자동화할 수도 있다. changeDictionary 기능은 system/ 디렉토리 하위의 영역 별 디렉토리에 저장되는 changeDictionaryDict 딕셔너리 파일을 이용해 설정한다.

아래는 heater 영역에 대한 changeDictionaryDict 파일 예시이다. 먼저, `boundary` 서브딕셔너리에서 이름이 `minY`, `minZ`, `maxZ`인 경계면의 타입이 `patch`로 설정된다. `T`는 영역 내부에 대해서는 300으로 균일하게 덮어쓰기되고, 경계면에 대해서는 별도의 설정이 입력된다. 먼저 경계면의 이름과 상관 없이 모든 경계(`".*"`)에 대해 `zeroGradient` 조건이 부여된다. 그 후, 이름이 `"heater_to_.*"` 패턴과 일치하는 경계면은 `turbulentTemperatureCoupledBaffleMixed` 타입으로 변경되고, 이름이 `minY`인 경계면은 `fixedValue` 타입으로 변경된다.

```cpp
//**************************************//
boundary
{
    minY
    {
        type patch; 
    }
    minZ
    {
        type patch;
    }
    maxZ
    {
        type patch;
    }
}
T
{
    internalField uniform 300;
    boundaryField
    {
        ".*"
        {
            type zeroGradient;
            value uniform 300;
        }
        "heater_to_.*"
        {
            type compressible::turbulentTemperatureCoupledBaffleMixed;
            Tnbr T;
            knappaMethod solidThermo;
            value uniform 300;
        }
        minY
        {
            type fixedValue;
            value uniform 500;
        }
    }
}
//**************************************//
```

meshQualityDict 파일에서 아래 줄을 수정하여 사용하라.

> **Note:** `#includeEtc "caseDicts/meshQualityDict"` $\hookrightarrow$ `#includeEtc "caseDicts/mesh/generation/meshQualityDict.cfg"`

> **Note:** OpenFOAM v1906: meshQualityDict 파일을 수정하지 않고 사용할 것.

> **Note:** OpenFOAM v1906: system/bottomAir 디렉토리의 fvSolution 파일에 `pRefCell`, `pRefValue`를 추가할 것.

```cpp
//**************************************//
PIMPLE
{
    momentumPredictor yes;
    nCorrectors 2;
    nNonOrthogonalCorrectors 0;
    pRefCell 0;
    pRefValue 1e5;
}
//**************************************//
```

> **Note:** OpenFOAM v1906: system/bottomAir, system/topAir 디렉토리의 fvSolution 파일에 세미콜론이 누락되어 있으니 추가하여 사용할 것.

```cpp
//**************************************//
"(rho|rhoFinal)"
{
    solver PCG;
    preconditioner DIC;
    tolerance 1e-7;
    relTol
}
//**************************************//
```

> **Note:** OpenFOAM v2412: meshQualityDict 파일을 수정하지 않고 사용할 것.

### 3.2 격자 생성 및 해석 실행

아래의 명령어로 `blockMesh`를 실행하여 배경 격자를 생성한다.

```bash
blockMesh
```

단일 영역 해석의 경우와 마찬가지로, `surfaceFeatures` 명령어를 이용해 \*.stl 형식의 형상 정보로부터 eMesh 파일을 생성한다. 또한, constant/ 디렉토리에는 extendedFeatureEdgeMesh/ 디렉토리가 생성된다. `surfaceFeatures` 명령어를 이용해 eMesh 파일을 생성하는 것은 필수적인 단계는 아니며, 특정 edge에 세분할이 필요한 경우에만 사용한다.

```bash
surfaceFeatures
```

> **Note:** OpenFOAM v1906: `surfaceFeatures` $\rightarrow$ `surfaceFeatureExtract`

> **Note:** OpenFOAM v2412: `surfaceFeatures` $\rightarrow$ `surfaceFeatureExtract`

격자 생성을 병렬 처리하기 위해서는 `snappyHexMesh`를 실행하기 전에 먼저 형상을 분할해야 한다. decomposeParDict 파일에 명시된 `numberOfSubdomains` 개수 만큼의 processor/ 디렉토리가 생성된다.

```bash
decomposePar
```

> **Note:** OpenFOAM v2412: `decomposePar` $\rightarrow$ `decomposePar -allRegions -force`

> **Note:** 영역 분할 시 `scotch` 방법을 사용하면 `snappyHexMesh`를 실행하거나 분할된 격자를 재구성하는 과정에서 오류를 일으킬 수 있어서 권장하지 않으며, 대신 `hierarchical`이나 `simple` 방법을 사용하는 것이 좋다.

Castellation, Snapping, Layering 등 각 격자 생성 단계마다 디렉토리가 생성되도록 하지 않고 최종적으로 생성된 격자만 저장할 때는 `-overwrite` 옵션을 사용하여 이전 격자 생성 단계의 데이터를 덮어쓴다.

```bash
mpirun -np 4 snappyHexMesh -parallel -overwrite
```

> **Note:** snappyHexMeshDict 파일에서 `castellatedMesh`와 `snap`이 true이고 `addLayer`가 false인 경우, 중간 단계인 `castellatedMesh`는 덮어쓰기되고 `snap` 단계의 격자만 저장된다. `castellatedMesh`, `snap`, `addLayer`가 모두 true인 경우, 중간 단계인 `castellatedMesh`, `snap` 격자는 모두 덮어쓰기되고, 마지막 단계인 `addLayer` 격자만 저장된다.

여기에서는 전체 영역에 대한 격자에 적용되는 단계이기 때문에, `castellatedMesh`와 `snap`만 true이고 `addLayer`는 false로 설정된다. 다음 명령어를 실행하여 최종 격자를 재구성한다.

```bash
reconstructParMesh -constant
```

`reconstructParMesh` 명령어를 실행하면 터미널에 아래와 같은 경고가 출력되는데, 격자 생성에 문제가 발생하지 않았다면 무시한다.

```text
This is an experimental tool, which tries to merge individual processor meshes back into one master mesh. ...
Not well tested & use at your own risk!
```

> **Note:** OpenFOAM v2412에서는 위 경고가 출력되지 않는다.

이 단계까지 모든 영역에 대해 격자가 생성되지만, 이 격자들은 영역 끼리 서로 연결되어 있으므로 분리 작업이 필요하다. 격자 생성 과정에서는 각 영역의 격자가 특정 플래그로 표시되며, 이 플래그를 기반으로 격자를 분할한다. 아래의 명령어를 이용하여 `cellZones`에 설정된 플래그를 기준으로 격자를 분할하고, 영역 별로 분할된 격자들은 각 영역의 polyMesh/ 디렉토리에 저장된다. 이 때, 해당 디렉토리에 기존 격자가 있는 경우에는 덮어쓴다.

```bash
splitMeshRegions -cellZones -overwrite
```

각 영역 별로 준비된 격자에 적절한 값을 부여한다. system 하위의 각 영역별 디렉토리의 changeDictionaryDict 파일을 이용해 설정하며, 따라서 아래 명령어를 각 영역 별로 반복하여 입력해야 한다.

```bash
changeDictionary -region heater
changeDictionary -region leftSolid
changeDictionary -region rightSolid
changeDictionary -region bottomAir
changeDictionary -region topAir
```

후처리를 위해, 고체 영역에서 유체 변수를 삭제한다. 작업 디렉토리의 Allrun 파일에 이 예제를 실행하기 위한 작업 흐름이 정리되어 있으며, 이를 참고한다.

솔버를 실행하여 계산을 시작한다.

```bash
chtMultiRegionFoam
```

```bash
mpirun -np 4 chtMultiRegionFoam -parallel > log
```

```bash
mpirun -np 128 --hostname machines chtMultiRegionFoam -parallel
```

### 3.3 후처리

해석 결과는 영역 별로 저장되어 있으며, 따라서 먼저 영역 별 해석 결과를 VTK 파일로 변환해야 한다.

```bash
foamToVTK -region heater
foamToVTK -region leftSolid
foamToVTK -region rightSolid
foamToVTK -region bottomAir
foamToVTK -region topAir
```

영역 별 해석 결과를 확인할 때는 `-region` 옵션을 이용한다.

```bash
paraFoam -region heater
...
paraFoam -region topAir
```

전체 영역에 대한 해석 결과를 확인할 때는 \*.foam 또는 \*.OpenFOAM 파일을 생성하여 ParaView로 불러들인다.

```bash
touch myCase.foam
paraview myCase.foam
```

![heater 영역의 온도 분포 (15초, 75초)](./figures/13_temp_heater.jpg)

![전체 영역의 온도 분포 (15초, 75초)](./figures/13_temp_entire.jpg)

![Color bar](./figures/13_temp_legend.jpg)
