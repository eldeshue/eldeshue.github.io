---
Date: 2025-03-21
tags:
  - Graphics
  - OpenGL
---
# Description
셰이더란 gpu의 각 코어에서 실행되는 프로그램. 

그래픽스 api 종류에 따라서 [[GLSL]] 혹은 HLSL로 문법이 다양하다.

정점 셰이더는 정점 데이터를 다룬다.

# Example

``` GLSL
#version 330 core 

layout (location = 0) in vec3 aPos;

void main()
{
	gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);
}

```
위는 하나의 정점을 처리하는 간단한 vertex shader이다.
```
#version 330 core 
```
먼저 첫 줄에 해당 shader의 OpenGL 버젼과 core profile 모드임을 명시한다.

```
layout (location = 0) in vec3 aPos;
```
그 다음 줄에는 vertex buffer의 layout을 작성하는데, 이는 shader로 하여금 vertex buffer에서 얼마만큼의 데이터를 읽어서 어떻게 번역할 지 정의하기 위함이다. 

위 코드를 기준으로 하면 해당 shader는 3개의 float을 입력으로 받으며, 이를 aPos라는 float 3개로 구성된 vector, vec3에 저장함을 의미한다. 
```
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec4 aColor;
```
위는 7개의 float을 입력으로 받는 shader의 입력을 정의한 것이다. 7개의 float 중, 앞의 3개를 x, y, z의 position으로 받고, 그 뒤의 4개 float에 대하여 x, y, z, w로 받았다.

```
void main()
{
	gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);
}
```
끝으로, main함수이다. input layout에서 받은 정점의 위치 데이터를 gl_Position에 할당하고 있다. gl_Position은 vec4 type의 데이터로, vertex shader의 output으로 미리 정의된 변수이다.  특히 네 번째 값인 w값은 view 좌표계로 변환된 정점 데이터를 NDC에 맞도록 좌표 값을 나눠주는 역할을 한다. 다만, 현재 w값이 1.0으로 설정되므로, 별도의 나눗셈 없이 그대로 전달된다.

즉, 위 shader는 그 어떤 처리도 하지 않고, 단순히 정점을 출력으로 전달만 한다.

일반적으로 vertex shader에서 행렬 변환(local to global, global to view)가 수행된다.