# 예제 12. 격자 생성 도구 (`snappyHexMesh` - 단일 영역)

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
   - 1.1 복잡한 형상에 대한 격자 생성 기법
   - 1.2 격자 생성 도구
2. 예제 개요
3. 해석
   - 3.1 전처리
   - 3.2 `snappyHexMesh` 실행
   - 3.3 격자 검사
   - 3.4 시뮬레이션 실행

## 1. 배경

이 예제에서는 OpenFOAM의 `snappyHexMesh` 도구를 사용한다. `snappyHexMesh`는 육면체(hexahedron)와 분할-육면체(split-hexahedron)로 구성된 3차원 격자를 생성하는 격자 생성 도구이다. 복잡한 형상에 대해 생성할 수 있는 격자의 종류를 살펴보고, `snappyHexMesh`를 다른 격자 생성 도구와 비교해본다.

### 1.1 복잡한 형상에 대한 격자 생성 기법

지금까지는 `blockMesh`를 이용하여 데카르트 좌표계에서만 격자 생성 작업을 해왔지만, 많은 공학 문제에서 데카르트 좌표계에는 정확히 맞아 떨어지지 않는 복잡한 형상을 해석해야 한다. 그런 경우에는 데카르트 좌표계를 이용한 격자 시스템보다는 곡률과 형상의 복잡성을 처리할 수 있는 다른 시스템을 사용하는 것이 더 유리하다.

복잡한 형상에 대한 격자 생성 기법은 크게 아래의 두 종류로 분류된다.

- 정렬 곡선 (structured curvilinear) 격자 배열
- 비정렬 (unstructured) 격자 배열

**정렬 격자**를 사용하는 경우,

- 격자 중심점이 좌표선의 교점에 위치한다.
- 한 격자에 인접한 격자(neighboring cell)의 개수가 일정하게 고정된다.
- 격자 중심점은 그 위치를 기반으로 행렬에 대응될 수 있다.
- 행렬에서 격자 중심점의 구조 및 위치는 인덱스로 표시된다. (2차원: I, J; 3차원: I, J, K)

매우 복잡한 형상인 경우, 해석 영역을 여러개의 블록으로 분할하여 각 블록마다 정렬 격자를 생성한다. 이를 **블록 정렬 격자(block-structured grids)**라 한다. 그 다음 수준의 복잡성에는 비정렬 격자를 사용한다. 비정렬 격자는 형상의 복잡성에 제한을 받지 않지만, 그 대신 프로그래밍 및 계산 비용을 더 요구한다. 최근에는 계산 자원의 발전으로 산업 분야의 CFD에서 비정렬 격자가 널리 활용된다.

### 1.2 격자 생성 도구

유$\cdot$무료의 다양한 격자 생성 도구가 있고, 주요 소프트웨어로는 ANSYS Meshing, ICEM, GAMBIT, Salome, `snappyHexMesh`, `cfMesh` 등이 있다. 여기서는 ANSYS GAMBIT, `snappyHexMesh`, `cfMesh`에 대해 자세히 설명한다.

#### 1.2.1 GAMBIT

GAMBIT은 3차원 비정렬 격자 생성 도구로, GAMBIT을 이용해 격자 생성 기법을 지정하려면 두 개의 파라미터를 특정해야 한다.

- **Element**
- **Type**

Element 파라미터는 해석 영역에 생성될 개별 격자의 형태를 정의하고, Type 파라미터는 정의된 element가 해석 영역 내에 생성되는 패턴을 정의한다. 단일 GUI 환경에서 3차원 형상 생성과 그 형상에 대한 격자 생성을 함께 진행한다.

#### 1.2.2 `snappyHexMesh`

형상 모델링과 격자 생성을 동시에 수행할 수 있는 GAMBIT과 달리, OpenFOAM에 내장된 `snappyHexMesh`는 미리 만들어진 형상에 대해 배경 격자(base mesh)를 생성하고, 이를 바탕으로 비정렬 격자를 생성한다. 배경 격자는 보통 `blockMesh`를 이용해 생성한다. `snappyHexMesh`의 대표적인 특징은 아래와 같다.

- 병렬 처리를 통한 빠른 격자 생성
- \*.stl, \*.obj 파일 지원
- 내부 및 벽면에 레이어 추가 기능
- 구획 별 격자 생성 (zonal meshing)

`snappyHexMesh`는 세 단계를 통해 격자를 생성하며, 각 단계는 아래와 같다.

- **Castellation**: 미리 정의된 지점을 포함하지 않는 영역(즉, 해석 영역 외부)에 생성된 배경 격자를 제거
- **Snapping**: 해석 영역 내부에 있는 격자 면/선을 경계면/선으로 이동시켜서 격자를 재구성
- **Layering**: 경계면/선에 격자 레이어 생성

다른 격자 생성 도구 대비 `snappyHexMesh`의 장점은 아래와 같다.

- 다른 상업용 소프트웨어를 필요로 하지 않는다. 격자 생성을 위해 OpenFOAM 환경으로 충분하다.
- 형상 모델링에 어떤 CAD 소프트웨어를 사용해도 된다. 형상은 면 정보만 필요로 하기 때문에, 형상 파일은 \*.stl, \*.nas, \*.obj 형식이어야 한다.
- 격자 생성 과정을 병렬로 처리할 수 있다. 계산 자원이 충분하다면 고 품질의 격자를 짧은 시간에 생성할 수 있다.

#### 1.2.3 `cfMesh`

`cfMesh`는 OpenFOAM에 구현되어있는 오픈소스 격자 생성 라이브러리로, 현재 2, 3차원의 사각/육면체 격자, 사면체 격자, 다면체 격자를 생성할 수 있다. `cfMesh`를 이용한 격자 생성은 mesh template - mesh modifier로 구성된 두 단계로 이루어진다. mesh modifier는 shared memory parallelization (SMP)와 distributed memory parallelization using MPI를 통한 병렬 처리가 가능하다.

## 2. 예제 개요

### 예제 구성

이 예제는 아래의 순서로 구성된다.

- 형상 모델링
- 단일 영역에 대한 격자 생성
- 생성된 격자에 대해 `scalarTransportFoam` 솔버를 이용한 OpenFOAM 시뮬레이션 실행

### 목표

- `snappyHexMesh`를 이용해 단일 영역에 격자를 생성하는 기본적인 과정을 이해한다.
- `snappyHexMesh`의 장점을 이해한다.
- `snappyHexMesh`의 격자 생성 과정 3단계를 이해한다.

### 데이터 처리

- 해석 결과를 ParaView로 확인하고, 플랜지 내부의 열 분포를 분석한다.

## 3. 해석

### 3.1 전처리

#### 3.1.1 예제 복사

아래의 예제를 작업 디렉토리로 복사한다.

```bash
$FOAM_TUTORIALS/mesh/snappyHexMesh/flange
```

#### 3.1.2 \*.stl 파일 구성

일반적으로 \*.stl 파일은 CATIA, freeCAD 등의 CAD 소프트웨어를 이용하여 생성한다. \*.stl 파일은 고체 형상의 정보를 가지고 있다. 이 예제에서는 OpenFOAM 예제 디렉토리에서 \*.stl 파일을 복사하여 사용한다. 아래의 \*.stl 파일을 작업 디렉토리의 constant/triSurface/ 디렉토리에 복사한다.

```bash
$FOAM_TUTORIALS/resources/geometry/flange.stl.gz
```

#### 3.1.3 constant/ 디렉토리

constant/ 디렉토리는 반드시 아래의 하위 디렉토리를 가지고 있어야 한다.

-- **triSurface**:

triSurface/ 디렉토리는 격자가 생성될 형상 정보가 포함된 파일(\*.stl, \*.nas, \*.obj 등)을 가지고 있어야 한다. 파일의 이름은 참조 포인터로 사용된다.

> **Note:** 모든 \*.stl 파일은 ascii 형식이어야 하며, **닫힌** 형상을 구성해야 한다.

#### 3.1.4 system/ 디렉토리

`snappyHexMesh`로 격자를 생성하기 위해서는 system/ 디렉토리 안에 아래의 파일이 있어야 한다.

\- **blockMeshDict**:

`snappyHexMesh`로 격자를 생성하기 위해서는 미리 생성된 배경 격자가 있어야 한다. 배경 격자는 반드시 형상 전체를 둘러싸고 있어야 한다. 배경 격자는 snappyHexMeshDict 파일에 정의된 설정에 따라 분할되고, 남은 부분은 제거된다. 배경 격자는 대개 `blockMesh`를 이용해 생성한다.

> **Note:** 날카로운 모서리를 적절하게 분할하기 위해, 배경격자를 정육면체로 구성하는 것이 좋다.

\- **decomposeParDict**:

`snappyHexMesh`를 이용하여 격자를 생성할 때, `decomposePar`를 이용하면 격자 생성 과정을 병렬로 처리할 수 있다. decomposeParDict 파일로 각 병렬 프로세서의 파라미터를 정의한다.

\- **meshQualityDict**:

격자 품질을 검사하기 위한 파라미터와, 생성된 격자의 각 파라미터 값이 이 파일에 저장된다.

\- **surfaceFeatureExtractDict**:

`snappyHexMesh`를 사용하기 전에 `surfaceFeatures`를 사용하면 날카로운 모서리 등을 인식하는데 도움을 주고, 해당 모서리에서의 격자 품질을 더 높일 수 있다. 한 모서리가 인접한 면과 이루는 각을 계산하고, 계산된 각이 surfaceFeaturesDict에 정의된 `includeAngle` 보다 작으면 그 모서리를 constant/extendedFeatureEdgeMesh/ 디렉토리의 \*.extendedFeatureEdgeMesh 파일에 따로 저장한다. 저장된 *날카로운 모서리*는 이후의 격자 생성 과정에서 처리된다.

```cpp
//**************************************//
Surfaces ("flange.stl");

includedAngle    150;
//**************************************//
```

> **Note:** OpenFOAM v1906: `surfaceFeatures` $\rightarrow$ `surfaceFeatureExtract`, surfaceFeaturesDict $\rightarrow$ surfaceFeatureExtractDict

```cpp
//**************************************//
flange.stl
{
    extractionMethod    extractFromSurface;
    
    extractFromSurfaceCoeffs
    {
        includedAngle   150;
    }
    writeObj            yes;
}
//**************************************//
```

\- **snappyHexMeshDict**:

이 파일은 `snappyHexMesh`를 실행하기 위해 필요한 설정을 입력한다. `snappyHexMesh`를 이용한 격자 생성 과정은 아래와 같이 3단계로 구성된다.

1. Castellating
2. Snapping
3. Layering

snappyHexMeshDict의 첫 부분에서 `castellatedMesh`, `snap`, `addLayers`를 true 또는 false로 설정하여 각 단계를 활성화/비활성화할 수 있다.

```cpp
//**************************************//
CastellatedMesh     true;
snap                true;
addLayers           true;
//**************************************//
```

`geometry` 딕셔너리에는 snappyHexMeshDict에서 사용되는 모든 면이 나열되고, 각각의 면을 참조할 때 쓸 이름이 함께 정의된다. 단, `blockMesh`에 의해 생성되는 면은 제외된다. 그 후, 계산 영역 중 격자를 세분할(refinement)하여 격자 품질을 개선하고자 하는 영역을 특정한다. 세분할 영역에는 임의의 이름을 부여하여 사용하고, 이 예제의 경우 refineHole이라는 이름으로 구의 반경과 중심 위치를 정의한다.

```cpp
//**************************************//
geometry
{
    flange
    {
        type triSurfaceMesh;
        file "flange.stl";
    }
    refineHole
    {
        type searchableSphere;
        centre (0 0 -0.012);
        radius 0.003;
    }
};
//**************************************//
```

> **Note:** OpenFOAM v1906: snappyHexMeshDict 파일의 구조가 아래와 같이 변경되었다.
>```cpp
>//**************************************//
>geometry
>{
>    flange.stl
>    {
>        type triSurfaceMesh;
>        file flange;
>    }
>    ...
>};
>//**************************************//
>```

> **Note:** OpenFOAM v2412: snappyHexMeshDict 파일의 구조가 아래와 같이 변경되었다.
>```cpp
>geometry
>{
>    flange.stl
>    {
>        type triSurfaceMesh;
>        name flange;
>    }
>    //- Refine a bit extra around the small centre hole
>    refineHole
>    {
>        type    sphere;
>        origin  (0 0 -0.012);
>        radius  0.003;
>    }
>}
>```

#### CASTELLATING

`castellatedMeshControls` 딕셔너리와 그 하위 딕셔너리는 castellating 단계에서 격자를 세분할할 때 적용할 파라미터를 정의한다.

Castellating 단계에서는 배경 격자를 더 작은 단위로 분할하고, 그 중 불필요한 부분을 제거한다. 배경 격자는 `level 0` 격자라고도 부른다. `level`을 1로 설정하면 배경 격자는 각 방향으로 절반으로 분할되어, 3차원의 경우 1개의 배경 격자가 8개로 분할된다. 즉, 각 레벨마다 격자의 개수는 8배로 증가한다.

![세분할 level](./figures/12_refinement.png)

- `features`: "\*.extendedFeatureEdgeMesh"로 지정된 *날카로운 모서리*에 적용할 세분할 level

- `refinementSurfaces`: 격자 면에 기반한 격자 세분할 level. 모든 격자면에 대해 2개의 `level` 값을 특정해야 한다. 첫 번째 값은 해당 면을 교차하는 모든 격자 셀에 적용되는 최소 세분할 level이고, 두 번째 값은 최대 세분할 level이다.

- `resolveFeatureAngle`: 모서리 세분할 기준 각도. 한 모서리에 인접한 면의 법선이 이 설정값보다 큰 각을 만들면 그 모서리는 한 단계 더 세분할된다. `resolveFeatureAngle` 값을 낮출 수록 *날카로운 모서리* 근처의 격자 품질이 좋아진다.

- `refinementRegions`: `geometry` 딕셔너리에서 정의한 세분할 영역(이 예제의 경우, `refineHole`)에서 격자를 세분할하는 정도를 정의한다. `levels`의 첫 번째 항목(1E15)은 해당 영역의 격자 세분할을 통해 생성될 격자의 최대 개수 제한이고, 두 번째 항목(3)은 세분할 `level`이다.

- `locationInMesh`: 형상의 내$\cdot$외부가 각각 어디인지를 정의한다. `locationInMesh` 지점이 포함된 영역이 계산 영역의 내부로 정의되며, 계산 영역의 **외부**에 생성된 격자는 제거된다. 단일 영역에 격자를 생성하는 경우에 `locationInMesh`를 계산 영역 내부로 정확히 지정하는 것이 중요하다.

```cpp
//**************************************//
castellatedMeshControls
{
    maxLocalCells       100000;
    maxGlobalCells      2000000;
    minRefinementCells  0;
    nCellsBetweenLevels 1;
    
    features
    {
        {
            file    "flange.extendedFeatureEdgeMesh";
            level   0;
        }
    };
    
    refinementSurfaces
    {
        flange
        {
            level (2 2);
        }
    }
    
    resolveFeatureAngle     30;
    
    refinementRegions
    {
        refineHole
        {
            Mode    inside;
            levels  ((1E15 3));
        }
    }
    locationInMesh (-9.23149e-05 -0.0025 -0.0025);
    allowFreeStandingZoneFaces true;
}
//**************************************//
```

> **Note:** `locationInMesh` 지점은 **격자 세분할이 완료된 이후에도** 격자면 위에 있어서는 안 된다. 어떤 경우에도 `locationInMesh` 지점은 격자 셀의 내부에 존재해야 하고, 그렇지 않으면 격자 생성에 실패한다.

Castellating 단계에서는 features, surfaces, regions에 대해 정의한 세분할 레벨에 따라 배경 격자를 세분할하고, 격자 셀 중 필요 없는 부분을 제거한다.

#### SNAPPING

`snapControls` 딕셔너리는 해석 영역 내부에 있는 격자 면/선을 경계면/선으로 이동시켜서 격자를 재구성할 때 적용할 파라미터를 정의한다.

- `nSolveIter`: 격자를 이동시킬 횟수

- `nFeatureSnapIter`: feature 모서리를 snapping할 횟수

```cpp
//**************************************//
snapControls
{
    nSmoothPatch            3;
    tolerance               1.0;
    nSolveIter              300;
    nRelaxIter              5;
    nFeatureSnapIter        10;
    implicitFeatureSnap     false;
    explicitFeatureSnap     true;
    multiRegionFeatureSnap  true;
}
//**************************************//
```

#### LAYERING

`addLayersControls` 딕셔너리와 그 하위 딕셔너리는 경계면/선에 레이어, 즉 격자층(boundary inflation layer)을 생성할 때 적용할 파라미터를 정의한다. 여기서 쓰이는 각 라벨은 constant/polyMesh/ 디렉토리의 boundary 파일에서 Boundary surface에 쓰인 라벨과 동일하다.

- `nSurfaceLayers`: 레이어의 개수

- `expansionRatio`: 인접 레이어 간 두께의 비율

- `finalLayerThickness`, `minThickness`: 레이어의 최종 두께 및 최소 두께

- `nLayerIter`: Layering의 반복 계산 횟수. snapping 및 layering이 충분히 부드럽게 이루어지지 않았다면 반복 계산 횟수를 늘려서 해결할 수 있다.

```cpp
//**************************************//
addLayersControls
{
    relativeSizes               true;
    layers
    {
        "flange_.*"
        {
            nSurfaceLayers      3;
        }
    }
    expansionRatio              1.005;
    
    finalLayerThickness         0.3;
    minThickness                0.25;
    nGrow                       0;
    featureAngle                30;
    nRelaxIter                  5;
    nSmoothSurfaceNormals       1;
    nSmoothNormals              3;
    nSmoothThickness            10;
    maxFaceThicknessRatio       0.5;
    maxThicknessToMedialRatio   0.3;
    minMedianAxisAngle          90;
    nBufferCellsNoExtrude       0;
    nLayerIter                  50;
    nRelaxedIter                20;
}
//**************************************//
```

> **Note:** 위의 snappyHexMeshDict 파라미터 중 플랜지 예제에서 쓰인 주요 사항에 대해서만 설명하였다.

### 3.2 `snappyHexMesh` 실행

아래의 명령어로 `blockMesh`를 실행하여 배경 격자를 생성한다.

```bash
blockMesh
```

blockMeshDict의 설정에 따라서 x, y, z 방향으로 각각 20, 16, 12개의 배경 격자가 생성된다.

![플랜지 해석을 위한 배경 격자](./figures/12_blockmeshforflange.jpg)

```bash
surfaceFeatures
```

> **Note:** OpenFOAM v1906: `surfaceFeatures` $\rightarrow$ `surfaceFeatureExtract`

> **Note:** OpenFOAM v2412: `surfaceFeatures` $\rightarrow$ `surfaceFeatureExtract`

단일 프로세서로 플랜지 형상에 격자를 생성하는 명령어는 아래와 같다.

```bash
snappyHexMesh
```

> **Note:** `snappyHexMesh`를 이용한 격자 생성 과정은 병렬 처리도 가능하다. 여러 개의 프로세서를 이용한 격자 생성에 대해서는 [예제 13](./13_격자-생성-도구-snappyHexMesh-다중-영역.md)을 참고하라.

`snappyHexMesh`를 이용해 격자를 생성하면 각 격자 생성 단계마다 디렉토리를 생성하고 생성된 디렉토리에 격자 파일을 저장한다. 예를 들어, `castellatedMesh`만 true로 지정하고 `snap`과 `addLayers`는 false로 지정하면 새로운 디렉토리는 하나만 생성된다. `snap`도 true로 지정된 경우에는 2개의 디렉토리가, 그리고 `addLayers`까지 true로 지정된 경우에는 총 3개의 디렉토리가 생성되고 각 디렉토리에는 polyMesh/ 디렉토리가 하나씩 생성된다.

![`snappyHexMesh` 실행 후 디렉토리 구조](./figures/12_snappyhexmeshfolder1.jpg)

각 단계마다 디렉토리가 생성되도록 하지 않고 최종적으로 생성된 격자만 저장할 때는 아래와 같이 overwrite 플래그를 사용하여 이전 격자 생성 단계의 데이터를 덮어쓴다.

```bash
snappyHexMesh -overwrite
```

![`snappyHexMesh -overwrite` 실행 후 디렉토리 구조](./figures/12_snappyhexmeshfolder2.jpg)

이 경우에는 polyMesh/ 디렉토리가 constant/ 디렉토리에만 1개 생성된다. 그러나, 때로는 overwrite 플래그를 사용하지 않고 `snappyHexMesh`를 실행하는 것이 유용하다. 각 격자 생성 단계를 별도로 저장하면 특정 단계에서 격자를 수정하고자 할 때 모든 단계를 다시 실행하지 않아도 되기 때문에 계산 시간을 줄일 수 있다.

### 3.3 격자 검사

생성된 격자의 상태를 검사하기 위해 `snappyHexMesh`의 덮어쓰기 기능을 해제하고, 생성된 격자 파일을 VTK 파일로 변환하여 ParaView로 불러온다.

```bash
foamToVTK
```

ParaView의 Time을 바꿔서 `snappyHexMesh`의 각 단계 별 격자 상태를 확인할 수 있다. 즉, Time 1은 castellating 단계의 격자이고 Time 2는 snapping, Time 3은 layering 단계가 적용된 격자이다.

![castellating 단계의 격자 (면 세분할 레벨 2)](./figures/12_flangeparaview1.jpg)

![castellating 단계의 격자 (면 세분할 레벨 3)](./figures/12_flangeparaview2.jpg)

![snapping 단계의 격자 (면 세분할 레벨 3)](./figures/12_flangeparaview3.jpg)

![layering 단계의 격자 (면 세분할 레벨 3)](./figures/12_flangeparaview4.jpg)

위 그림들은 플랜지의 단면에서 격자를 나타낸 것으로, 각각 Castellating 단계 격자 (`refinementSurfaces (2, 2)`), Castellating 단계 격자 (`refinementSurfaces (3, 3)`), Snapping 단계 격자(`refinementSurfaces (3, 3)`), Layering 단계 격자(`refinementSurfaces (3, 3)`)이다. 단면의 위치는 아래의 그림에 붉은색 평면으로 나타내었다.

![플랜지의 단면 위치](./figures/12_flangeparaviewsection.jpg)

격자의 품질은 `checkMesh` 도구로 확인할 수 있다.

```bash
checkMesh
```

### 3.4 시뮬레이션 실행

#### 3.4.1 예제 복사

이 예제에서는 `scalarTransportFoam` 솔버를 이용하여 플랜지 내부 열 분포 해석을 수행한다. 솔버 설정을 위해 아래의 예제 파일을 작업 디렉토리로 복사한다.

```bash
$FOAM_TUTORIALS/basic/scalarTransportFoam/pitzDaily
```

플랜지 격자 파일을 작업 디렉토리로 이동시켜야 한다. 플랜지 디렉토리에서 가장 마지막 시간 단계의 polyMesh/ 디렉토리를 pitzDaily/ 디렉토리의 constant/ 디렉토리로 복사한다. `snappyHexMesh` 실행 시 덮어쓰기 옵션을 선택했다면 플랜지 디렉토리의 constant/polyMesh/ 디렉토리를 pitzDaily/ 디렉토리의 constant/ 디렉토리로 복사한다.

#### 3.4.2 케이스 설정

플랜지 내부 열 분포 해석을 위해 아래의 설정값을 변경하여야 한다. 먼저 0/ 디렉토리의 T 파일을 수정하여 플랜지의 초기 온도를 293K로 설정하고 입구로부터 350K로 가열되도록 설정한다.

```cpp
//**************************************//
dimensions          [0 0 0 1 0 0 0];

internalField       uniform 293;

boundaryField
{
    flange_patch1
    {
        type        fixedValue;
        value       uniform 350;
    }
    flange_patch2
    {
        type        fixedValue;
        value       uniform 293;
    }
    flange_patch3
    {
        type        fixedValue;
        value       uniform 293;
    }
    flange_patch4
    {
        type        fixedValue;
        value       uniform 350;
    }
}
//**************************************//
```

0/ 디렉토리의 U 파일에서 계산 영역 전체와 모든 경계면의 속도를 0으로 설정한다. system/ 디렉토리의 controlDict 파일에서 `endTime`을 0.0005로, `deltaT`를 0.000001로, `writeInterval`을 100으로 설정한다.

#### 3.4.3 솔버 실행

아래의 명령어로 `scalarTransportFoam` 솔버를 실행한다.

```bash
scalarTransportFoam
```

#### 3.4.4 해석 결과

```bash
paraFoam
```

![플랜지 내부 온도 변화 (0.0001--0.0005초)](./figures/12_flangeresult.jpg)
