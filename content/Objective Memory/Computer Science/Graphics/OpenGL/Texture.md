---
Date: 2025-04-02
tags:
  - Graphics
---
# Description
텍스처란 이미지로, polygon으로 된 3D 오브젝트에 씌우는 것으로 물체의 외형을 정의하는데 사용한다.

1D, 2D, 3D 등 여러 종류가 존재하지만, 기본적으로 2D이다.

2D 텍스처는 이미지이기에 튜플의 2차원 배열이지만, 이 자료구조에 이미지가 아닌 다른 데이터를 저장할 수도 있다.
## Sampling
3D 물체는 점의 집합이고, 이 점이 모여서 구성하는 평면에 텍스처를 씌우게 된다. 3D 오브젝트는 오직 점의 정보만 알 뿐, 그 점과 점 사이의 정보는 보간으로 이루어진다. 따라서 보간으로 생성된 평면의 각 위치에 텍스처의 색을 입히기 위해서는 평면을 구성하는 점이 텍스처의 어떤 위치에 해당하는지 알아야 할 필요가 있다. 즉, 텍스처 이미지의 좌표와 3D 오브젝트를 구성하는 점의 좌표 사이의 매핑이 필요하다. 이후 점과 점 사이에 존재하는 픽셀의 색은 매핑된 관계에 따라 보간으로 결정된다.

일반적으로 텍스처의 좌표는 0~1 사이로 scaling되어 float으로 표현된다.

텍스처의 좌표에 대응하는 색을 텍셀(Texel)이라 한다.

매핑 결과를 바탕으로 텍셀의 데이터를 가져와 픽셀을 결정하는 것을 샘플링이라 한다.  
## Minification
텍스처 축소. 물체에 상대적으로 고화질(많은 개수의 텍셀로 구성된) 텍스처를 씌우는 경우에 적용한다.

이는 **하나의 픽셀에 여러 텍셀이 대응되는 상태로, 다수의 텍셀이 버려지게 된다**. 이는 픽셀의 불연속성을 강조하여 렌더링한 이미지의 품질에 악영향을 끼친다.

이러한 경우 기존의 텍셀을 선형 보간해서 만들어낸 가상의 텍셀로 부터 샘플링을 수행한다. 그 결과, 해당 픽셀이 참조해야 하는 여러 텍셀의 색을 거리에 따른 가중치를 적용하여 함께 반영할 수 있다.
## Magnification
텍스처 확대. 물체에 상대적으로 저화질(적은 개수의 텍셀로 구성된) 텍스처를 씌우는 경우에 적용한다.

이는 **다수의 픽셀이 동일한 텍셀로 부터 샘플링을 거치게 되며**, 이 또한 픽셀의 불연속성을 강조하여 렌더링한 이미지의 품질에 악영향을 끼친다. 

이러한 경우 기존의 텍셀을 선형 보간해서 만들어낸 가상의 텍셀로 부터 샘플링을 수행한다. 즉, 불연속이 발생하는 영역을 부드럽게 뭉개준다.

# Texture in OpenGL
OpenGL에서 텍스처 관련하여 지원하는 여러 기능에 대해서 알아본다.
## Creating Texture Object
기타 OpenGL 오브젝트와 마찬가지로, 다음과 같이 생성하고 핸들을 획득한다. 이후 glTexImage2D를 호출하여 2D 텍스처를 이미지로 초기화 한다. 이 때 사용하는 이미지는 이미지 로더 등으로 획득한다.
``` C
// create texture
GLuint hTexture = 0;
glGenTextures(1, &hTexture);

/*
	매개변수 설명
	1st : 초기화 할 대상, 현재 GL_TEXTURE_2D로 bind 된 대상을 지정
	2nd : 생성할 밉맵의 레벨, 특정 레벨의 밉맵만 만들 경우에 값 전달, 그 외는  0으로 설정
	3rd : 생성될 텍스처의 색 형식, 이미지가 RGB 3개의 color값이면 GL_RGB
	4th : 이미지의 가로 너비
	5th : 이미지의 세로 높이
	6th : legacy, 항상 0
	7th : 이미지의 형식, 이미지 파일이 3종의 색으로 구성됨을 알림
	8th : 이미지의 형식, 각 색깔이 unsigned char, 즉 1Byte로 제공됨을 알림
	9th : 이미지의 데이터의 포인터
*/
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, data); 

glGenerateMipmap(GL_TEXTURE_2D); // 현재 bind 된 텍스처로 밉맵 생성

```

## Applying Texture
텍스처를 적용하는 방법은 간단하다. 텍스처는 bind하면 shader에서 uniform의 형태로 접근할 수 있다. 이 때, fragment shader에 정점 좌표와 그에 대응하는 texture의 좌표를 함께 전달한 다음, texture의 좌표로 sampling을 수행하여 fragment의 색을 결정하면 된다. 

중요한 점은 vertex의 좌표와 그에 대응하는 texture의 좌표를 매핑해줘야 한다는 것이다. 이 부분만 만족된다면, vertex 이와의 부분은 이후 설명할 여러 기술에 의하여 보간되어 해결될 것이다.
## Texture Wrap
텍셀의 좌표는 0~1로 정규화 되어있다. 그러나, 매핑 과정에서  정규화된 좌표를 벗어나는 값이 필요할 수 있다. 이 때 사용하는 옵션이 바로 래핑이다. 래핑은 텍스처 주변의 텍셀을 결정하는 방법으로, OpenGL은 다음의 4가지 방법을 제공한다.

-  GL_REPEAT : 기존 텍스처가 단순하게 반복된다. 
- GL_MIRRORED_REPEAT : 기존 텍스처가 경계선을 기준으로 선대칭되어 반복된다.
- GL_CLAMP_TO_EDGE :  기존 텍스처의 경계선의 텍셀이 무한히 확장된다.
- GL_CLAMP_TO_BORDER :  기존 텍스처와 별개로 사용자가 지정한 텍셀의 값이 추가된다.
## Texture Filter
샘플링 과정에서 수행되는 오브젝트와 텍스처 이미지 사이의 차이를 보정하는 기법. 

슈퍼 샘플링과 유사한 효과를 내는데, 차이점이 있다면 슈퍼 샘플링은 샘플링을 다수 수행하는 것이고, 텍스처 필터링은 각각의 샘플링을 어떻게 수행하는지에 대한 것이다.
### Code
``` C
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST); // 축소 
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR); // 확대
```
OpenGL은 크게 두 옵션을 제공하는데, Nearest는 default 옵션으로, 좌표상 가장 가까운 텍셀을 대푯값으로 샘플링하는 방법이다. 그에 반해, Linear는 주변 픽셀에 대한 거리 기반의 선형 보간을 수행하여 샘플링을 수행한다.
## Mipmap
pre scaled texture. 메모리를 투자하여 미리 scaled 된 texture를 저장하고, 이 scaled 된 texture를 활용하여 filtering 등을 절약하는 테크닉. 

밉맵이란 고해상도 텍스처에서 발생하는 **미니피케이션**에 대응하기 위해서, 여러 사이즈(레벨)의 축소된 텍스처를 생성하여 저장해두고, 샘플링 시 필터링 대신 적절한 텍스처를 선택하여 대신 사용하는 방법이다. 

일반적으로 2의 거듭제곱으로 축소된 텍스처를 생성하기에 O(N)의 메모리를 추가적으로 소모하지만, 선형 보간 등 runtime에 수행할 계산을 절약할 수 있다. 또한 기존의 고해상도 texture 대신 밉맵으로 생성된 저해상도 텍스처를 사용하므로, cache 효율도 증가한다. 

밉맵으로 생성된 여러 레벨의 텍스처에 대해서, 이들 사이에서도 보간이 필요하다. 모든 상황에 대응하는 레벨의 텍스처를 생성할 수는 없기 때문이다. 이와 관련하여 OpenGL은 다음의 옵션을 제공한다.
- GL_NEAREST_MIPMAP_NEAREST : 샘플링은 near, 텍스처를 near level
- GL_NEAREST_MIPMAP_LINEAR : 샘플링은 near, 텍스처를 인접한 두 level의 linear
- GL_LINEAR_MIPMAP_NEAREST : 샘플링은 linear, 텍스처는 near level
- GL_LINEAR_MIPMAP_LINEAR : 샘플링은 linear, 텍스처를 인접한 두 level의 linear

이는 다음과 같이 적용될 수 있다.
``` C
// only works for minification
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
```
