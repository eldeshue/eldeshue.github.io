---
Date: 2025-03-26
tags:
  - Graphics
  - OpenGL
  - ProgrammingLanguage
---
# Description
GLSL(GL Shader Language)은 OpenGL에서 사용하는 shader를 정의하기 위한 프로그래밍 언어이다.

## What is Shader?
셰이더란 gpu의 각 core에서 실행되는 프로그램을 뜻한다. 셰이더는 그래픽스 파이프라인의 특정 영역에서 실행되며, 각각의 셰이더는 완벽하게 독립되어 존재한다. 즉 완전한 병렬성을 보장한다.

셰이더는 기본적으로 입력을 받아서 출력을 생성하는 프로그램이다. 셰이더의 입력 및 출력은 자유롭게 정의될 수 있으며, 이러한 자유로움으로 인해서 OpenGL 사용자는 input layout을 명시적으로 정의하는 불편함을 겪게 된다. 

그러나, 사용자가 정의 가능한 셰이더의 등장은 그래픽스 프로그래밍의 큰 발전을 가져왔으며, 이는 더 나아가 GPGPU의 등장으로 이어졌다.
# Syntax
GLSL은 셰이더를 작성에 특화된 프로그래밍 언어로, C와 유사한 문법을 지녔다. 또한, 행렬 및 벡터 연산을 위한 문법을 내장하고 있기에 상당히 편리하다.

gpu의 구현과 api의 명세를 독립적으로 유지하기 위해서 shader는 하드웨어에 종속되며, 따라서 최초 한 번의 실행에는 compile이 필요하다. 
## Type
GLSL은 C와 유사한 문법을 가진 강한 타입 언어이다. 기본적인 원시 타입(int, uint, float, double, bool)을 지원하며, 두 종류의 container인 벡터와 행렬을 제공한다.
### Vector
GLSL의 벡터는 그 보유하는 원소의 개수에 따라서 vec2, vec3, vec4의 3종이 존재한다. 기본적으로는 float을 갖지만, 접두어로 i, u, d, b가 붙는 겻에 따라서 int, uint, double, bool의 벡터가 되기도 한다.
벡터는 다음과 같이 초기화될 수 있다. 한 벡터는 다른 벡터의 인자로 복사될 수 있다.
``` GLSL
vec2 vect = vec2(0.5, 0.7);
vec4 result = vec4(vect, 0.0, 0.0); 
vec4 otherResult = vec4(result.xyz, 1.0);
```

벡터의 각 원소는 x, y, z, w 또는 r,g,b,a 혹은 s,t,p,q의 필드로 접근이 가능하다.
#### swizzling
GLSL의 벡터는 다음과 같은 독특한 문법인 swizzling을 지원한다.
``` GLSL
vec2 someVec; 
vec4 differentVec = someVec.xyxx; 
vec3 anotherVec = differentVec.zyw; 
vec4 otherVec = someVec.xxxx + anotherVec.yxzy;
```
swizzling이란 '.' 이후에 여러 필드를 이어서 쓰는 것으로, 임시 벡터를 만드는 문법이다. 이 때 임시로 생성된 벡터의 원소는 '.' 이후에 씌여진 원소를 순서대로 복사하여 구성된다.

#### Operator for vector
벡터는 다음과 같은 여러 연산을 지원한다.

## Basic Structure
``` GLSL
#version version_number

in type in_variable_name;
in type in_variable_name;

out type out_variable_name;

uniform type uniform_name;

void main()
{

	// process input(s) and do some weird graphics stuff 
	... 
	// output processed stuff to output variable 
	out_variable_name = weird_stuff_we_processed;
}

```
### Version
OpenGL은 하드웨어에 종속되고, 이는 셰이더도 마찬가지이다. 따라서, 해당 shader의 버젼을 명시하여 현재 실행 환경이 호환이 되는지 확인하는 작업이 필수적이다. 그러므로 shader 코드의 최상단에는 항상 OpenGL의 version을 명시한다.
```
versioin 330 core
```
위는 OpenGL 3.3 version임을 의미한다. 또한, 해당 OpenGL의 실행 환경이 core profile 모드라는 점도 명시를 해줘야 한다.
### Input and Output
모든 shader는 파이프라인의 이전 셰이더의 출력을 입력으로 전달 받는다. 따라서 이전 shader의 out 변수는 다음 shader의 in 변수와 일치해야 하며, 그렇지 않은 경우 shader를 링킹 하는 과정에서 문제가 생길 것이다. **여기서 일치한다 함은 type과 변수의 name이 모두 같음을 의미한다.**

다만, vertex shader와 fragment shader는 특별하다.
#### Vertex Shader - Input
vertex shader의 경우에는 vram에 적재된 buffer object에서 gpu의 컨트롤러가 정점 데이터를 공급한다. 따라서 vertex shader의 경우 특별히 다음과 같은 layout을 지정하며, 이를 vertex attribute라 한다. glVertexAttribPointer에서 명시하는 바로 그것이다.
``` GLSL
layout(location = 0) in vec3 aPos;
```
이러한 vertex layout의 경우, 하드웨어에 따라서 그 갯수에 한계가 있는데, 보통 16 * 4를 보장한다.

#### Fragment Shader - Output
fragment shader는 픽셀 후보의 색상을 결정한다. 따라서 반드시 vec4로 rgba 값을 출력으로 제공해야 한다.
``` GLSL
out vec4 fragColor; 
```

### Uniform
유니폼이란 shader의 input/output으로 전달되는 programmable한 입출력이 아닌, OpenGL 내부적으로 참조하는 일종의 전역변수다. 즉 input/output이 인자와 return value에 대응된다면, uniform은 람다의 capture에 해당한다. uniform의 선언은 다음과 같다.
``` GLSL
uniform vec3 test;
```

Uniform을 설정하기 위해서는 OpenGL에서 정의하는 특정 api를 호출해야만 하는데, 이는 다음과 같다.
``` GLSL
int testUniformLocation = glGetUniformLocation(hShaderProgram, "test");
glUseProgram(hShaderProgram); 
glUniform4f(testUniformLocation, 0.0f, 0.0f, 0.0f, 1.0f);
```
먼저 셰이더에서 선언한 uniform의 메모리 위치를 다음과 같이 획득한다.
```
int testUniformLocation = glGetUniformLocation(hShaderProgram, "test");
```
이 때, "변수명"을 전달하여 그 위치를 획득함을 주의하자.

다음으로는 해당 shader program을 사용하겠음을 gpu에 알린 후, 해당 shader에 선언된 uniform의 값을 초기화한다. 
``` GLSL
// load shader program
glUseProgram(hShaderProgram); 

// set value to the uniform variable
glUniform4f(testUniformLocation, 0.0f, 0.0f, 0.0f, 1.0f);
```
초기화 하려는 uniform data의 타입에 따라 glUniform 함수를 달리 호출해야 한다. 예제에서는 vec4이므로, glUniform4f를 호출했다. 즉 타입과 인자의 개수에 따라 다른 postfix를 갖는다. 

유니폼의 진정한 의의는 pipeline에서 정의된 입력/출력으로 단순 전달되는 in/out과는 별개로 참조가 가능하다는 점이다. 따라서 global view matrix 등, runtime에 update되는 render resource를 전달하기에 적절하다.

다만, 이러한 uniform의 경우에는 특별하게 명시적으로 정의된 데이터만 조작할 수 있으므로, 사용에 제한이 있다.

