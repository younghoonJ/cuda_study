# CUDA 스레드 계층
    스레드  < 워프 < 블록 < 그리드


### 워프
- 32개의 스레드를 하나로 묶은 단위. 
- cuda의 기본 수행 단위.
- 하나의 제어 장치로 제어됨.


### 블록
- 하나의 블록에 속한 스레드는 고요한 스레드 번호를 가짐
- 각 블록은 고유한 번호(block ID)를 가짐.
- 원하는 스레드를 지칭하려면 블록 번호와 스레드 번호 모두 필요.
- 1, 2, 3차원 형태로 배치 가능.


### 그리드
- 여러 개의 블록을 포함하는 블록들의 그룹.
- 1, 2, 3차원 형태로 배치 가능.
- 하나의 그리드에 속한 블록들은 고유한 block ID를 가짐.
- 커널 호출시 그리드 생성됨. 하나의 그리드는 하나의 커널 호출과 1:1 대응


## 내장 변수
    스레드들은 자신이 처리할 데이터가 무엇인지 알아야 함.
- gridDim: 그리드의 형태 정보.
- blockIdx: 현재 스레드가 속한 블록의 번호.
- blockDim: 블록의 형태 정보.
- threadIdx: 블록 내에서 현재 스레드가 부여받은 스레드 번호.


## 스레드 번호와 워프의 구성
- 워프는 연속된 32개 스레드로 구성됨
- 스레드 연속성은 threadIdx의 x,y,z 차원 순으로 결정됨. (0,0,0)~(31,0,0)번 스레드가 하나의 워프를 구성. 
- 만약 x차원의 길이가 워프의 크기보다 작다면 y차원의 번호가 낮은 순으로 연속성을 가짐. 만약 x차원이 1이면 (0,0,0)~(0,31,0) 순서.


## 그리드의 크기
- 최대 3차원. 
- x: 2^31 - 1
- y,z: 65535


## 블록의 크기
- x,y: 1024
- z: 64
- 블록 하나는 최대 1024개의 스레드까지 가질 수 있음.


## 스레드 레이아웃 설정 및 커널 호출
- Kernel<<<그리드 형태, 블록의 형태>>>()
- Dim3 구조체 사용.
```c
Dim3 dimGrid(4, 1, 1);
Dim3 dimBlock(8, 1, 1);
kernel<<<dimGrid, dimBlock>>>();
```


## 블록 내 스레드 전역 번호
- 1차원 블록: threadIdx.x == 스레드 전역 번호
- 2차원 블록: blockDim.x * threadIdx.y + threadIdx.x
- 3차원 블록: blockDim.x * blockDim.y * threadIdx.z + blockDim.x * threadIdx.y + threadIdx.x

## 그리드 내 스레드의 전역 번호
- 자신이 속한 블록의 앞 블록까지의 스레드 수 + 자신이 속한 블록 내에서 자신의 번호
- NTHREAD_PER_BLK = blockDim.z * blockDim.y * blockDim.x
- 1차원 그리드: NTHREAD_PER_BLK * blockIdx.x + TID_IN_BLK
- 2차원 그리드: NTHREAD_PER_BLK * (gridDim.x * blockIdx.y + blockIdx.x) + TID_IN_BLK
- 3차원 그리드: NTHREAD_PER_BLK * (gridDim.x * gridDim.y * blockIdx.z + gridDim.x * blockIdx.y + blockIdx.x) + TID_IN_BLK


## 그리드 -> GPU
- 쿠다 커널 실행시 스레드 레이아웃에 따라 그리드 생성.
- 그리드가 GPU를 사용하는 단위.


## 스레드 블록 -> SM
- 그리드에 포함된 스레드 블록은 그리드가 배정된 GPU 속 SM에 의해 처리됨.
- SM의 가진 자원의 양과 한 스레드 블록을 처리하기 위해 필요한 자원의 양에 따라 한 SM이 동시에 처리할 수 있는 스레드 블록 수가 결정됨.
- active block: SM에 할당된 블록 중 현재 필요한 자원을 모두 할당받고 실행할 수 있는 상태인 블록.

## 워프 & 스레드 -> SM 속의 CUDA 코어
- 스레드 블록에 포함된 스레드들은 워프로 분할.
- 워프는 32개의 스레드로 구성. 각 스레드는 쿠다 코어 하나에서 처리.
- 페르미 아키텍처의 경우 SM내 cuda 코어의 수가 32이며 16개씩 두 그룹으로 분리되어 있음. 그리고 각 쿠다코어 그룹이 하나의 워프를 처리.(아키텍처별 다름)
- 워프는 하나의 명령어로 움직임. 즉, 워프 속 스레드들은 하나의 명령어로 제어됨.

### 스레드의 실행 문맥
- 워프 속 스레드는 하나의 명령어로 제어되지만 스레드의 실행 문맥은 독립적이며 레지스터로 관리됨. (SIMD)와 다름.
- 스레드 블록 내 모든 워프가 SM 내부 레지스터 파일을 나누어 사용함.
- SM 안에 있는 여러 개의 워프가 서로 번갈아 cuda 코어를 사용함.
- 문맥 교환 비용이 0에 가까워 GPU 알고리즘의 경우 코어 수보다 적게는 3~4배 많게는 10개 이상의 스레드를 사용하기를 권장함.
- 스레드가 자신만의 실행 문맥을 가지지만 명령어는 같으므로 워프 내 스레드들이 분기가 있으면 순차적으로 처리해야 함.


# CUDA 메모리 계층
## 스레드 수준 메모리(per-thread memory)
- 레지스터
    - Cuda 코어 연산을 위한 데이터를 담아두고 사용하는 공간. 스레드가 연산을 위해 데이터를 저장하는 공간. 커널 내부에서 선언된 지역 변수를 위해 사용됨. 
    - SM 내부에 있는 메모리 (in-core memory). 1GPU cycle 이내 접근.
    - 블록 또는 SM 하나당 8~64K개의 32비트(4바이트) 레지스터를 가짐. 스레드 블록 내 모든 스레드들은 내부의 레지스터를 나누어 사용. 즉, 하나의 스레드 블록이 최대 스레드 수 (1024)개를 사용하는 경우, 한 스레드는 8~64개의 레지스터만 사용 가능. 활성 블록수가 두 배가 되면 스레드 당 할당 가능한 레지스터의 수가 절반으로 줄어 듦
- 지역 메모리(local memory)
    - SM 밖에 있는 오프-칩 메모리(off-chip memory). 400~600 GPU cyles 정도에 접근. 물리적으로는 GPU의 디바이스 메모리(DRAM 영역) 공간 일부가 지역 메모리로 사용됨.
    - 레지스터 공간을 할당받기에 큰 구조체나 배열 또는 일반 변수가 레지스터 공간을 할당받지 못하는 경우 지역 메모리 공간을 사용.
    - 스레드당 512KB 제한이 있지만 스레드 하나가 지역 변수를 위해 사용하기에는 충분함.

## 블록 수준 메모리(per-block memory)
- 블록 내 모든 스레드들이 접근할 수 있는 공유 메모리 공간.
- 물리적으로 SM 내부에 위치. 1~4 GPU cycles
- compute capability에 따라 16~96KB의 크기를 가짐.
- 크기는 작지만 접근 속도가 매우 빠른 공유 메모리 공간을 잘 사용해야 알고리즘의 성능이 빠름.

### 정적 할당(static allocation)
```
__global__ void kernel(void)
{
    __shared__ int sharedMemory[512];
}
```
- 각 스레드의 지역 변수가 아닌 스레드 블록 내 모든 스레드가 공유하는 변수로 선언. 스레드 블록당 하나만 선언됨.
- 정적 할당의 경우 컴파일 시에 크기가 결정.

### 동적 할당(dynamic allocation)
```
extern __shared__ int sharedPool[];
int *sIntArray = sharedPool;
fload *sFloatArray = (float*)&sharedPool[sizeIntArr];

__global__ void kernel(void)
{
    ...
    sIntArray[threadIdx.x] = 0;
    sFloatArray[threadIdx.x] = 0.0f;
    ...
}

int main(void)
{
    int size = 512;
    kernel <<<gridDim, blockDim, 
    sizeof(int) * sizeIntArr + sizeof(float) * sizeFloatArr>>>();
}
```
- 크기가 정해지지 않은 extern 배열 형태(빈 대괄호, []사용)로 커널 밖에서 선언.
- 커널 호출시 세 번째 인자 값으로 배열의 크기 전달.
- 세 번째 인자에는 하나의 값만 전달 가능하므로 여러 개의 공유 메모리 공간을 동적 할당할 수 없음. 만약 하나의 커널 안에서 여러 개의 공유 메뫃리 배열이 필요하다면 모든 배열의 크기를 더한 만큼의 공간을 가지는 하나의 큰 공유 메모리 배열을 선언하고, 포인터를 이용하여 해당 공간을 분할


## 그리드 수준 메모리(per-grid memory)
- 커널을 수행하는 모든 스레드가 접근 가능.
- 전역 메모리, 상수 메모리, 텍스처 메모리. 세 종류 모두 디바이스 메모리 공간 사용.
- 전역 메모리는 읽기/쓰기 가능. 나머지는 read-only.
- 상수 메모리와 텍스처 메모리는 특수 목적 메모리이며, 온-칩 캐시(on-chip cache)를 활용함.

 
### 전역 메모리
- 모든 스레드에서 접근 가능한 디바이스 메모리 공간.
- 400~800 GPU cycles 정도로 가능 느림.
- 스레드 수준 및 블록 수준 메모리와 달리 호스트에서 접근 가능한 GPU 메모리.

### 상수 메모리
```
__constant__ int constMemory[512];

__global__ void kernel(void)
{
...
int a = constMemory[threadIdx.x]; // Okay
...
}

int main(void)
{
    int table[512] = {0};
    cudaMemcpyToSymbol(constMemory, table, sizeof(int) * 512);
    ...
    kernel <<<gridDim, blockDim>>>();
}

```
- 최대 64KB. 디바이스 메모리 영역을 사용하며, 전용 온-칩 메모리인 상수 캐시(constant cache)를 사용함. 상수 캐시의 크기는 compute capability에 따라 다르며 대략 48KB 수준. 즉, 상수 메모리에 대한 접근은 캐싱되며, 캐시 힛이 높은 경우 접근 속도가 크게 높아짐.
- 연산에 필요한 참조 테이블 등 자주 접근하면서 커널 수행 중 값이 변하지 않는 데이터의 경우, 상수 메모리를 활용하면 CUDA 프로그램의 성능 향상의 효과를 기대할 수 있음.
- `__constant__`를 통해 전역 범위로 선언. 그 값은 호스트에 의해 커널 호출 전에 초기화 되어야 함. `cudaMemcpyToSymbol` 사용.

### 텍스처 메모리
- 그래픽 관련 연산을 위해 사용되는 공간. 2차원 공간적 지역성에 최적화 되어 있으며, floating-point interpolation과 같은 다양한 하드웨어 필터링을 지원함.





### 



