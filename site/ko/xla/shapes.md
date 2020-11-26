# 형상 및 레이아웃

XLA `Shape` proto([xla_data.proto](https://www.tensorflow.org/code/tensorflow/compiler/xla/xla_data.proto))는 N 차원 배열(간단히 *배열*)의 랭크, 크기 및 데이터 유형을 설명합니다.

## 용어, 표기법 및 규칙

- The rank of an array is equal to the number of dimensions. The *true rank* of an array is the number of dimensions which have a size greater than 1.

- Dimensions are numbered from `0` up to `N-1` for an `N` dimensional array. The dimension numbers are arbitrary labels for convenience. The order of these dimension numbers does not imply a particular minor/major ordering in the layout of the shape. The layout is determined by the `Layout` proto.

- 관례적으로, 차원은 차원 번호의 오름차순으로 나열됩니다. 예를 들어, 크기가 `[A x B x C]`인 3차원 배열의 경우, 차원 0은 크기가 `A`이고, 차원 1은 크기가 `B`이며, 차원 2는 크기가 `C`입니다.

    XLA의 일부 유틸리티는 Python과 유사하게 음의 인덱싱도 지원합니다. 차원 -1은 마지막 차원입니다(`N` 차원 배열의 경우 `N-1`과 동일). 예를 들어 위에서 설명한 3차원 배열의 경우, 차원 -1의 크기는 `C`이고 차원 -2의 크기는 `B`입니다.

- 2차원, 3차원 및 4차원 배열에는 종종 차원과 관련된 특정 문자가 있습니다. 2D 배열을 예로 들면 다음과 같습니다.

    - 차원 0: `y`
    - 차원 1: `x`

    3D 배열의 경우:

    - 차원 0: `z`
    - 차원 1: `y`
    - 차원 2: `x`

    4D 배열의 경우:

    - 차원 0: `p`
    - 차원 1: `z`
    - 차원 2: `y`
    - 차원 3: `x`

- 차원을 취하는 XLA API의 함수에서는 차원 번호가 오름차순으로 적용됩니다. 이것은 `initializer_list`로 차원을 전달할 때 사용되는 순서와 일치합니다. 예를 들어, 아래와 같은 경우

    `ShapeUtil::MakeShape(F32, {A, B, C, D})`

    차원 크기 배열이 시퀀스 `[A, B, C, D]`로 구성된 형상이 만들어집니다.

## 레이아웃

`Layout` proto는 배열이 메모리에서 어떻게 표현되는지 설명합니다. `Layout` proto에는 다음 필드가 포함됩니다.

```
message Layout {
  repeated int64 minor_to_major = 1;
  repeated int64 padded_dimensions = 2;
  optional PaddingValue padding_value = 3;
}
```

### 마이너-메이저 차원 순서 정하기

유일한 필수 필드는 `minor_to_major`입니다. 이 필드는 형상 내에서 차원의 마이너-메이저 순서를 설명합니다. `minor_to_major` 값은 가장 작은 차원인 첫 번째 값부터 가장 큰 차원인 마지막 값까지 배열 차원의 순서입니다(`N` 차원 배열의 경우 `0`에서 `N-1`까지). 가장 작은 차원은 선형 메모리에 배치된 배열 요소를 단계별로 실행할 때 가장 빠르게 변경되는 차원입니다.

예를 들어, `[2 x 3]` 크기의 다음 2D 배열을 생각해보겠습니다.

```
a b c
d e f
```

여기서 차원 `0`은 크기가 2이고, 차원 `1`은 크기가 3입니다. 레이아웃의 `minor_to_major` 필드가 `[0, 1]`이면 차원 `0`이 가장 작은 차원이고 차원 `1`이 가장 큰 차원입니다. 이것은 선형 메모리의 다음 레이아웃에 해당합니다.

```
a d b e c f
```

`0`에서 `N-1`까지의 이 마이너-메이저 차원 순서는 *열-메이저*(랭크 2)와 유사합니다. 단조 차원 순서를 가정하면 코드에서 이 레이아웃을 참조하는 데 사용할 수 있는 또 다른 이름은 단순히 "dim 0 is minor"입니다.

한편, 레이아웃의 `minor_to_major` 필드가 `[1, 0]`이면 선형 메모리의 레이아웃은 다음과 같습니다.

```
a b c d e f
```

`N` 차원 배열에 대해 `N-1`에서 `0`까지의 마이너-메이저 차원 순서는 *행-메이저*(랭크 2)와 유사합니다. 단조로운 차원 순서를 가정하면 코드에서 이 레이아웃을 참조하는 데 사용할 수 있는 또 다른 이름은 단순히 "dim 0 is major"입니다.

#### 기본 마이너-메이저 순서

The default layout for newly created Shapes is "dimension order is major-to-minor" (akin to row-major at rank 2).

### 채우기

채우기는 선택적 `padded_dimensions` 및 `padding_value` 필드에 정의됩니다. `padded_dimensions` 필드는 각 차원이 채워지는 크기(너비)를 설명합니다. 이 필드가 있는 경우, `padded_dimensions`의 요소 수는 형상의 랭크와 같이야 합니다.

예를 들어, 위에 정의된 `[2 x 3]` 배열에서 `padded_dimensions`가 `[3, 5]`이면 차원 0은 너비 3으로 채워지고, 차원 1은 너비 5로 채워집니다. 선형 메모리의 레이아웃(0의 채우기 값과 열-메이저 레이아웃 가정)은 다음과 같습니다.

```
a d 0 b e 0 c f 0 0 0 0 0 0 0
```

이는 마이너-메이저 차원 순서가 같은 다음 배열의 레이아웃과 동일합니다.

```
a b c 0 0
d e f 0 0
0 0 0 0 0
```

### 배열로 인덱싱

`IndexUtil`의 [IndexUtil](https://www.tensorflow.org/code/tensorflow/compiler/xla/index_util.h) 클래스는 형상과 레이아웃이 입력되면 다차원 인덱스와 선형 인덱스를 상호 변환하는 유틸리티를 제공합니다. 다차원 인덱스에는 각 차원에 대한 `int64` 인덱스가 포함됩니다. 선형 인덱스는 배열을 보유하는 버퍼로 인덱싱하는 단일 `int64` 값입니다. 형상과 레이아웃의 생성 및 조작을 단순화하는 이 유틸리티에 대해서는 같은 디렉토리에 있는 `shape_util.h` 및 `layout_util.h`를 참조하세요.