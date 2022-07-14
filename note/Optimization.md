# 최적화에 관한 자세한 내용

※ 기술적인 내용으로 가득 차 있다. 컴퓨터에 관한 지식이 있다면 읽는 데에 도움이 된다.

서술한 내용 중 JIT, OpenGL, NumPy 등의 최적화 방안을 모두 종합하여 라이브러리를 구성한 뒤 테스트를 진행한 결과,   
다음과 같은 결과를 얻었다.

- **1920 x 1080** 크기의 전체화면 창에서 **끊김 없이 실시간으로 모든 픽셀 갱신 가능** 

## 방안 1 ─ JIT(Just-In-Time) Compile

인터프리터 언어의 특징적인 속도 저하 문제를 해결하고자 도입할 방안.   
프로그램 실행 시점에 일부 코드를 네이티브 코드로 컴파일하고,   
해당 코드를 프로그램에서 재사용할 때 인터프리터가 재해석하여 실행할 필요가 없도록 하여 큰 폭의 성능 향상을 꾀한다.

JIT 방식으로 컴파일될 함수는 후술할 GPGPU의 **커널**과 비슷하게 함수 호출을 비롯한 대부분의 기능에 제약이 생긴다.
```py
def jitted_function(arg1, arg2, arg3):
  other_function() # 호출할 수 없음
  for i, v in enumerate(arg1): #이런 기본적인 연산 위주로만
    arr[i] = v + 1 # 가능
```
JIT은 상당한 성능 향상을 가져올 수 있지만, 위 코드처럼 제약이 생겨나기 때문에 코드의 추상화가 어렵고,    
문법적 제약이 생기며 하드코딩이 반강제된다는 단점을 지닌다.

<img src="https://user-images.githubusercontent.com/34784356/171444243-eb635934-838d-4c02-a436-02bfa8510699.png" height="150">   

현재 프로젝트에서 쓸 자체 제작 그래픽 라이브러리 **Pi**는   
후술할 OpenGL로 렌더링할 버퍼(화면에 표시할 픽셀 정보를 담는 배열)에 관한 대규모 연산을 수행할 수 있도록 하는 함수를 제공하는데,   
이 과정에서 버퍼를 이리저리 변형시키고 다룰 함수를 전달해야 하는(Callback function) 상황이 생겨난다.    
JIT 컴파일된 코드에서 콜백 함수를 실행시키도록 하면 오버헤드가 발생하여 성능이 하락하므로, 오버헤드를 막기 위해 다음과 같은 스타일을 사용하게 된다.     
```py
def jitted_process(index, value): # 실제로 버퍼를 조작할 함수
  return randint(0, 255)
iterator = buffer_iteratoriv(jitted_process) # 버퍼 조작 함수를 최초 1회 생성하도록 하는 생성 함수 호출

new_buffer = iterator(buffer) # 버퍼 조작 및 교체
```
그러나 이러한 코드 구성은 여러 불편함을 가져오기에 개선할 필요가 있어 더 나은 방안을 연구해볼 예정이다.   
## 방안 2 ─ GPGPU(General-Purpose computing on GPU, GPU 범용 연산)

CPU의 연산 효율을 극복하기 위한 방안.   

<img src="https://user-images.githubusercontent.com/34784356/172621002-1db1aa72-9bed-4c60-8e6f-364367ef1d66.png" width="430">   

CPU와 GPU는 **산술 논리 장치(ALU)** 를 통해 각종 연산을 수행하는데, GPU가 CPU에 비해 압도적으로 많은 수를 가지고 있다.

JIT 컴파일을 통해 네이티브 코드를 실행함으로써 비약적인 속도 향상을 얻어낼 수 있지만, 
CPU의 연산만으로 짧은 시간 내에 처리가 불가능한 대규모의 연산이 필요한 경우 GPU를 연산에 사용하여 효율을 높이도록 구성한다.   

현재 프로젝트에서는 NVIDIA에서 제공하는 **CUDA(Compute Unified Device Architecture)** API를 활용하여 **커널**(GPU에서 실행될 함수)을 작성하고    
이를 통해 주로 크기가 큰 버퍼 관련 연산을 처리하도록 구성해볼 계획이다.   

커널은 외부에 의존적일 수 없고, 여러 문법들을 지원하지 않으며, 값을 반환할 수 없다는 특징을 가지고 있어   
어느정도 편하게 사용이 가능하도록 구성한다고 해도 편의성 손실은 감안해야 한다.

## 방안 3 ─ OpenGL

<img src="https://user-images.githubusercontent.com/34784356/171444466-28829294-1281-4c80-8766-6b31daab42cc.png" width="250px">   


**OpenGL**은 그래픽 렌더링을 담당하는 산업용 저수준(Low-level) API로,  
하드웨어 가속(GPU)을 사용하는 만큼 속도를 보장할 수 있게 된다.   

액체 시뮬레이터는 화면의 픽셀을 직접 조작하도록 구성할 가능성이 높은데,   
PyGame과 같은 라이브러리는 후처리(Post-Processing)를 수행하기에 부적합하므로 OpenGL을 기반으로 렌더링 엔진을 설계하도록 한다.

## 방안 4 ─ NumPy

<img src="https://user-images.githubusercontent.com/34784356/172631564-2e684532-13c4-4c27-af39-76cb7a538ecb.png" width="200px"/>   

**NumPy**는 대부분이 **C**로 작성된, 수학 관련 연산 처리에 적합한 라이브러리이다.   

제아무리 JIT을 활용하여 최적화를 수행한다고 해도, 다음과 같은 상황에서는 매우 치명적인 시간 소모가 일어나게 된다.

```py
import random
def jitted_randomize(buffer):
  for e in buffer:
    buffer[e] = random.randint(0, 255)
  return buffer
```   

그래서 배열 관련 연산 등을 최적화하기 위하여 C로 짜여진 NumPy를 적극 활용하도록 한다.

```py
from numpy.random import randint

def jitted_randomize(buffer):
  for e in buffer:
    buffer[e] = randint(0, 255)
  return buffer
```   

## 방안 5 ─ C/C++ Bindings

복잡하고 규모가 큰 연산을 파이썬에게 맡기는 대신,   
연산 담당 함수를 C/C++로 작성하고 파이썬에서 이를 호출할 수 있도록 구성하여 성능 향상을 꾀한다.   
대표적으로 **ctypes**, **boost**, **cppyy** 등을 사용하여 C/C++ 바인딩을 구현할 수 있다.

## 방안 6 ─ ChildProcess Spawning

마찬가지로 복잡하고 규모가 큰 연산을 파이썬에게 맡기는 대신,   
Node.js로 짜인 프로그램과 같이 파이썬에 비해 빠른 속도를 낼 수 있는 프로세스를 호출하여 일부 과정을 담당하도록 한 뒤,   
`stdout` 등을 이용하여 연산 결과를 가져오는 방법이다.   
※ - Node.js는 1억 번 while을 돌리는 동일한 과정에서 파이썬보다 약 111배 빠른 것으로 측정되었다.
