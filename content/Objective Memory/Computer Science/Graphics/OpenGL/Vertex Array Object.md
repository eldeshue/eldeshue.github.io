---
Date: 
tags:
  - Graphics
  - OpenGL
---
# Description
앞서 정의한 [[Vertex Buffer Object]]에 관련한 설정 정보를 별도로 저장, 관리하는 객체, [[Vertex Buffer Object]] 및 [[Element Buffer Object]]의 handle(id)과 glVertexAttribPointer, glEnableVertexAttribArray 등의 설정을 저장한다. 

VAO는 설정을 대신 관리할 뿐 아니라, bind하는 것으로 gpu의 render 관련 설정을 한 번에 변경한다.
## Background of VAO
OpenGL의 구조는 State Machine이다. 따라서 우리는 이전에 어떤 대상을, 어떻게 그려야 하는지, 관련 데이터는 어디서 읽어야 하는지 등의 정보를 매 render 마다 수동으로 정의해줘야 한다. 이러한 반복은 대단히 불편하다. 따라서 이러한 문제를 해결하기 위해서 VBO와 관련된 설정을 모아서 관리하는 객체가 바로 VAO(Vertex Array Object)이다.

VAO의 도입을 통해서 비로소 instancing을 비롯한 객체지향적 접근을 할 수 있게 되었다.
## Relation between VAO and VBO - View

VAO는 snap-shot이 아니다. VAO는 VBO의 memory address가 아닌 handle을 저장하므로, VBO의 변화는 VAO에 반영된다. 따라서 VAO는 gpu의 state에 대한 capture 혹은 view라 해야 할 것이다.

다만, VAO의 life time은 VBO에 종속된다. 따라서 VBO가 먼저 소멸한다면, VAO는 더 이상 유효하지 않다. 
# Related Functions
``` C

// construction
GLuint hVAO;
glGenVertexArrays(1, &hVAO);

glBindVertexArray(hVAO);	// begin, bind VAO
/*
	do something...
	bind vertex buffer, bind element buffer, etc...
*/
glBindVertexArray(0);

glDeleteVertexArrays(1, &hVAO);

```

## glGenVertexArrays - construct
마치 buffer object 생성과 마찬가지로, 해당 함수를 호출하고 핸들을 받는다.
## glDeleteVertexArrays - destruction
마치 buffer object 생성과 마찬가지로, 핸들을 받아서 삭제한다.

## glBindVertexArray - decorator
VAO의 초기화 및 사용은 마치 decorator처럼 동작한다.

VBO, EBO, attribPointer, enable/disable Attrib 의 앞에서 hVAO를 인자로 호출하면 capture을 시작한다. 이후 0으로 호출하면 capture를 종료한다.

마찬가지 방법으로, gpu의 상태를 복원할 때, 복원할 VAO의 핸들을 전달하여 호출하고, 제거할 때, 인자를 0으로 호출한다.