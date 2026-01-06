---
Date: 2025-03-25
tags:
  - Graphics
  - OpenGL
---
# Description
cpu와 gpu는 서로 다른 프로세서이고, 별도의 메모리를 사용한다. 일반적으로 렌더링의 자원이 될 데이터(3D 모델, 텍스처, 등)는 main memory에 있으므로,  cpu는 gpu에게 이를 제공할 필요가 있다. 이를 적재한다고 표현하며, 과정에서 둘 사이에서 IO가 발생한다. 

빈번한 IO호출은 효율에 좋지 않은 관계로, 메모리가 허락하는 범위 내에서 한 번에 최대한 많은 양을 전달하는 것이 바람직한데, 이를 위한 것이 바로 여러 buffer object이다.  VBO는 그 중 정점 데이터를 저장한다.

이들 BO는 gpu의 빠른 접근을 위해서 gpu의 메모리에 위치한다.
# Related Functions

``` C++
GLuint hVBO;
glGenBuffers(1, &hVBO);	// buffer object 생성

// gpu에게 앞으로 설정할 GL_ARRAY_BUFFER 관련 설정의 대상을 지정(bind)
glBindBuffer(GL_ARRAY_BUFFER, hVBO);

// 앞서 array buffer로 선언한 buffer object에 데이터 적재, cpu와 gpu 사이의 transcation
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices.data(), GL_STATIC_DRAW);

// buffer object를 vertex buffer로 정의
// vertex shader의 입력으로 동작하기 위한 memory layout을 정의
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(Point), (void*)0);	

// vertex shader의 입력 인자로 활성화
glEnableVertexAttribArray(0); 

// delete buffer
glDeleteBuffers(1, &hVBO);
```

## glGenBuffers - Construction 
gpu에 버퍼 오브젝트를 생성하고, 그 handle을 가져온다. 이 때 생성된 buffer object에는 타입이 없고, 이후 사용 방식에 따라서 그 정의가 결정된다.
## glBindBuffer - Declare Target
이후 glBindBuffer를 호출하여 작업하고자 하는 버퍼를 지정한다. glBindBuffer는 gpu의 상태를 변경하여 이후에 호출할 buffer와 관련된 상태 변경을 hVBO에게 수행할 것을 지정한다.

glBufferData를 비롯한 이후의 거의 모든 함수는 BO의 핸들을 인자로 받지 않는데, glBindBuffer에서 지정했기에 그런 것으로, gpu가 이미 조작 대상을 알고 있기 때문이다.
## glBufferData - GPU Transcation
Main Memory의 데이터를 gpu의 vram에 적재한다. 이 때 전달하는 데이터는 pointer와 size의 형태로 표현된다. 

이 때 마지막 인자로 usage, 즉 어떤 메모리에 적재할 지에 대한 힌트를 받는데, 이는 다음과 같다.
- GL_STREAM_DRAW: constant, temporary data
- GL_STATIC_DRAW: constant, long term used data
- GL_DYNAMIC_DRAW: mutable(frequent write), long term used data

cpu와 gpu 사이의 transaction이므로, 그 비용이 상당하다. 따라서 빈번한 사용은 비용이 될 수 있다.
이 usage는 vram내의 적재 위치에 대한 힌트라고 할 수 있다.
## glVertexAttribPointer - Define Input Layout
VBO의 존재 목적은 shader에 vertex 관련 데이터를 전달함에 있다. 따라서 shader program의 정의에 맞게 VBO를 연결해야 하는데, 이를 지정하는 과정이 바로 glVertexAttribPointer 이다.
``` GLSL
// example of input layout of vertex shader
layout (location = 0) in vec3 aPos;
```

해당 함수는 다음의 인자를 받는다.
1. locaiton : vertex shader가 정의한 입력 중 몇 번째에 해당하는지 지정한다. 위 예시에서는 location이 0이므로, 0을 전달한다.
2. number of element in vertex : aPos의 타입이 vec3이므로, 3
3. type of element in vertex : vec3를 구성하는 각 type이 float이므로 GL_FLOAT
4. normalize : NDC로 정규화 되지 않은 정수 등을 VBO로 삼은 경우, 정규화를 할 것인지 명시
5. stride : 다음 데이터와의 거리 차이, 보통 연속적인 데이터의 경우 vertex data의 크기를 준다
6. offset of buffer : buffer를 읽기 시작하는 위치를 결정

이러한 설정 값의 전달을 통해서 gpu로 하여금 shader program에 data를 적절하게 공급할 수 있도록 정의한다. 

첫 번째 인자가 location이므로, 매 layout마다 별도로 호출해야 함.
## glEnableVertexAttribArray - Enable Layout
일반적으로 shader의 layout은 사용하지 않음으로 비활성화 되어있다. 이를 활성화 해줘야 gpu가 정상적으로 해당 layout에 data를 공급한다.

인자가 location이므로, 매 layout마다 별도로 호출해야 함.
## glDeleteBuffers - Destruction
끝으로 glDeleteBuffers를 호출하여 gpu메모리에서 buffer object를 삭제한다.