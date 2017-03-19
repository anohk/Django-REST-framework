# Renderers
REST framework에는 다양한 미디어 타입으로 응답을 리턴할 수 있는 여러가지 Renderer 클래스가 포함되어있다. 또한 커스텀 렌더러를 정의할 수 있다. 

## How the renderer is determined
view에 대한 유효한 renderer set은 항상 클래스 목록으로 정의된다. view가 입력되면  REST framework는 들어오는 요청에 대한 content negotiaion을 수행하고 요청을 만족시키는 데에 가장 적합한 렌더러를 결정한다. 

content negotiation의 기본 프로세스는 요청의 Accept 헤더를 조사하여 응답에서 예상하는 미디어타입을 판별하는 것이다. 선택적으로 URL 형식의 접미사를 사용해 명시적으로 특정 표현을 요청할 수 있다. 예를 들어 URL `http://example.com/api/users_count.json`은 항상 JSON data를 반환하는 엔드포인트가 될 수 있다. 

더 자세한 내용은 [content negotiation](http://www.django-rest-framework.org/api-guide/content-negotiation/) 참고

## Setting the renderers
기본 renderer set은 `DEFAULT_RENDERER_CLASSES`설정을 사용해 전역으로 설정할 수 있다. 예를들어 다음 설정은 `JSON`을 기본 미디어타입으로 사용하며 자체 기술 API도 포함한다. 

```python
REST_FRAMEWORK = {
	'DEFAULT_RENDERER_CLASSES': (
		'rest_framework.renderers.JSONRenderer',
		'rest_framework.renderers.BrowsableAPIRenderer',
	)
}
```

APIView 클래스 기반 view를 사용해 개별 view 또는 viewset에 사용되는 renderer를 설정할 수 있다. 

```python
from django.contrib.auth.models import User
from rest_framework.renderers import JSONRenderer
from rest_framework.response import Response
from rest_framework.views import APIView

class UserCountView(APIView):
	renderer_classes = (JSONRenderer,)
	
	def get(self, request, format=None):
		user_count = User.objects.filter(active=True).count()
		content = {'user_count': user_count}
		return Response(contet)
```

`@api_view`데코레이터를 사용하는 함수 기반 view에서는 다음과 같다. 

```python
@api_view(['GET'])
@renderer_classes((JSONRenderer,))
def user_count_view(request, format=None):
	user_count = User.objects.filter(active=True).count()
	content = {'user_count':user_count}
	return Response(content)
```

## Ordering of renderer classes
API의 renderer class를 지정해 각 미디어타입에 할당할 우선순위를 고려할 때 중요하다. 클라이언트가 `Accept: */*` 헤더를 보내는 것과 `Accept` 헤더를 포함하지 않는 것과 같이 받아들일 수 있는 표현을 클라이언트가 명시하지 않으면 REST framework는 목록에서  첫 번째 renderer를 선택해 응답에 사용한다. 

예를들어 API가 JSON 응답과 HTML 탐색이 가능한 API를 제공하는 경우 Accept 헤더를 지정하지 않은 클라이언트에 `JSON` 응답을 보내려면 `JSONRenderer`를 기본 렌더러로 설정하는 것이 좋다. 

API 요청에 따라 일반 웹페이지와 API응답을 모두 제공할 수 있는 view가 포함된 경우, [broken accept headers](http://www.newmediacampaigns.com/blog/browser-rest-http-accept-headers)를 보내는 오래된 브라우저에서 제대로 작동하기 위해, 기본 렌더러를 `TemplateHTMLRenderer`로 만드는 것을 고려해야한다. 

# API Reference
## JSONRenderer
utf-8 인코딩을 사용하여 요청 데이터를 `JSON`으로 렌더링한다. 

기본 스타일은 유니코드 문자를 포함하고, 불필요한 공백없이 컴팩트 스타일을 사용해 응답을 렌더링한다. 

```python
{"unicode black star": "★", "value":000}
```

클라이언트는 `indent` 미디여 타입 매개변수를 추가로 포함할 수 있다. 이 경우 리턴된 `JSON`은 들여쓰기된다. 

예시: `Accept: application/json; indent=4`

```python
{
	"unicode bloack star": "★",
	"value": 999
}
```

기본 JSON 인코딩 스타일은 `UNICODE_JSON`및 `COMPACT_JSON`설정 키를 사용해 변경할 수 있다. 

**.media_type:** `application/json`  
**.format:** `.json`  
**.charset:** `None`