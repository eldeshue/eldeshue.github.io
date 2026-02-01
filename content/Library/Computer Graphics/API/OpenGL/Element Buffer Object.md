---
Date: 2025-03-25
tags:
  - Graphics
  - OpenGL
---
# Description

EBO(Element Buffer Object)는 Buffer Object의 일종으로, vertex의 index를 저장한다. 
vertex buffer와 함께 사용하여 render resource를 정의한다.

EBO는 VBO와 마찬가지로 [[Vertex Array Object]]에게 capture될 수 있는데, glDrawElements를 사용하면 EBO를 참조하여 렌더링을 수행할 수 있다.

보통 rendering 과정에서, 하나의 정점 자원을 복사하지 않고 여러번 사용하기 위해서 도입한다.

DirectX에서도 index buffer의 개념이 존재하기에 여러 하드웨어에서 공통된 원리로 동작한다고 생각된다.
# Related Functions
``` C

GLuint hEBO;
glGenBuffers(1, &hEBO);	// create buffer for EBO

// bind VAO before bind EBO
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, hEBO);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices.data(), GL_STATIC_DRAW);

```
EBO는 일반적인 buffer object이므로, 생성과 소멸 및 초기화는 VBO와 동일하다. 

달리 말하면 Buffer Object에 어떤 속성을 bind 하느냐가 중요하다.

VBO는 buffer를 bind할 때 GL_ARRAY_BUFFER에 bind하지만, EBO는 GL_ELEMENT_ARRAY_BUFFER에 bind한다. 

EBO를 bind했다면, 이는 render 시 indexbuffer를 참조하길 바란 것이다.  따라서, glDrawElements를 호출해야 한다.