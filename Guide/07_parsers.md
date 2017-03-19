# Parsers
REST framework에는 Parser 클래스가 내장되어 있어 다양한 미디어타입으로 요청을 받을 수 있다. 또한 커스텀 parser를 정의할 수 있어 API에서 허용하는 미디어타입을 유연하게 디자인할 수 있다. 


## How the parser is determined
view에 대한 유효한 parser set은 항상 클래스의 목록으로 정의된다. `request.data`에 접근하면 REST framework는 들어오는 요청의 `Content-Type`헤더를 검사하고 요청내용에 어떤 parser를 사용할지 결정한다. 


**Note:**  
클라이언트 응용 프로그램을 개발할 때는 항상 HTTP 요청으로 데이터를 보낼 때 `Content-Type` 헤더를 설정해야한다. 

content type을 설정하지 않으면 대부분의 클라이언트는 `application/x-www-form-urlencoded`를 기본값으로 사용한다. 

예를들어 `.ajax()` 메서드로 jQuery를 사용해 `json`으로 인코딩 된 데이터를 보내는 경우 `contentType: 'application/json'` 설정을 해줘야한다. 

## Setting the parsers
기본 parser set은 `DEFAULT_PARSER_CLASSES` 설정을 이용해 전역에서 사용할 수 있다.  
예를들어 다음의 설정은 기본 JSON 또는 form data 대신 `JSON` 콘텐츠가 있는 요청만 허용한다. 

```python
REST_FRAMEWORK = {
	'DEFAULT_PARSER_CLASSES': (
		'rest_framework.parsers.JSONParser',
	)
}
```
APIView 클래스 기반 view를 사용해 개별 view 또는 viewset에 사용되는 parser를 설정할 수도 있다. 

```python
from rest_framework.parsers import JSONParser
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
	parser_classes = (JSONParser,)
	
	def post(self, request, format=None):
		return Response({'received data': request.data})
```

함수 기반 view에서 `@api_view`데코레이터를 사용하는 경우에는 다음과 같다. 

```python
from rest_framework.decorators import api_view
from rest_framework.decorators import parser_classes

@api_view(['POST'])
@parser_classes((JSONParser,))
def example_view(request, format=None):
	return Response({'received data': request.data})
```

# API Reference
## JSONParser
`JSON` 요청 내용을 분석한다. 

**.media_type:** `application/json`

## FormParser
HTML form 내용을 분석한다. `request.data`는 데이터의 `QueryDict`로 채워진다. 

일반적으로 HTML form data를 완벽히 지원하기 위해 `FormParser`와 `MultiParser`를 함께 사용한다. 

**.media_type:** `application/x-www-form-urlencoded`

## MultiPartParser
파일 업로드를 지원하는 멀티파트 HTML form 내용을 분석한다. 두 `request.data` 모두 `QueryDict`로 채워진다. 

일반적으로 HTML form data를 완벽히 지원하기 위해 `FormParser`와 `MultiParser`를 함께 사용한다. 

**.media_type:** `multipart/form-data`

## FileUploadParser
raw file 업로드 내용을 분석한다. `request.data` 속성은 업로드 된 파일을 포함하는 단일 키 `file` 이 포함된 dictionary이다. 

`FileUploadParser`와 함께 사용된 view가 `filename` URL 키워드 argument로 호출되면 해당 argument가 filename으로 사용된다. 

`filename` 키워드 argument 없이 호출되면 클라이언트는 `Content-Disposition` HTTP header에 filename을 설정해야한다. 예시: `Content-Disposition: attachment; filename=upload.jpg`

**.media_type:** `*/*`

**Notes:**  

- `FileUploadParser`는 파일을 raw 데이터 요청으로 업로드할 수 있는 기본 클라이언트에서 사용하기 위한 것이다. 웹 기반 업로드 또는 멀티 파트 업로드가 지원되는 기본 클라이언트의 경우 `MultiPartParser` 파서를 사용해야한다. 
- 이 파서의 `media_type`은 모든 콘텐츠 타입과 일치하기때문에 `FileUploadParser`는 일반적으로 APIView 에 설정된 유일한 파서여야한다. 
- `FileUploadParser` 는 Django의 표준 `FILE_UPLOAD_HANDLERS`설정과 `request.upload_handlers` 속성을 고려한다.  자세한 내용은 [Django documentation](https://docs.djangoproject.com/en/1.10/topics/http/file-uploads/#upload-handlers)


```python
# example

# views.py
class FileUploadView(views.APIView):
	parser_classes = (FileUploadParser,)
	
	def put(self, request, filename, format=None):
		file_obj = request.data['file']
		...
		return Response(status=204)

# urls.py
urlpatterns = [
	...
	url(r'^upload(?P<filename>[^/]+)$', FileUploadView.as_viw())
]
```

# Custom parsers
커스텀 파서를 구현하려면 `BaseParser`를 오버라이드하고, `.media_type` 속성을 설정하고, `.parse(self, strea, media_type, parser_context`) 메서드를 구현해야한다. 

메서드는 `requset.data` 속성을 채우는데 사용할 데이터를 리턴해야한다. 

`.pares()`에 전달 된 argument는 다음과 같다. 
### stream
요청의 본문을 나타내는 객체같은 stream이다. 

### media_type
선택적 argument. 들어오는 요청 콘텐츠의 미디어타입이다. 

요청의 `Content-Type:` 헤더에 따라서 렌더러의 `media_type`속성보다 구체적일 수 있다. media_type 매개변수가 포함될 수 있다. 예: `"text/plain; charset=utf-8"`

### parser_context
선택적 argument. 해당 argument가 주어지면 요청 내용을 분석하는데 필요한 추가적인 컨텍스트를 포함하는 dictionary가 된다. 

기본적으로 키에는 `view`, `request`, `args`, `kwargs`가 포함된다. 

### Example
다음은 요청 본문을 나타내는 문자열로 `request.data` 속성을 채우는 일반 텍스트 파서의 예시이다. 

```python
class PlainTextParser(BaseParser):
	media_type = 'text/plain'
	
	def parse(self, stream, media_type=None, parser_context=None):
	return stream.read()
```


# Third party packages

## YAML
[REST Framework YAML](http://jpadilla.github.io/django-rest-framework-yaml/)은 [YAML](http://www.yaml.org/) 분석과 렌더링을 제공한다.

### Installation & configuration

```python
pip install djangorestframework-yaml
```

```python
# settings.py
REST_FRAMEWORK = {
	'DEFAULT_PARSER_CLASSES': (
		'rest_framework_yaml.parsers.YAMLParser',
		),
	'DEFAULT_RENDERER_CLASSES': (
		'rest_framework_yaml.renderers.YAMLRenderer',
	),
}
```
## XML
[REST Framework XML](http://jpadilla.github.io/django-rest-framework-xml/)은 간단한 비공식 XML  형식을 제공한다. 

### Installation & configuration

```python
pip install djangorestframework-xml
```

```python
# settings.py
REST_FRAMEWORK = {
	'DEFAULT_PARSER_CLASSES': (
		'rest_framework_mxl.parsers.XMLParser',
		),
	'DEFAULT_RENDERER_CLASSES': (
		'rest_framework_xml.renderers.XMLRenderer',
	),
}
```
## MessagePack
[MessagePack]()은 빠르고 효율적인 이진 직렬화 형식이다. [Juan Riaza](https://github.com/juanriaza)는 MessagePack 렌더러와 REST framework에 대한 파서 지원을 제공하는 [djangorestframework-msgpack](https://github.com/juanriaza/django-rest-framework-msgpack) 패키지를 유지관리한다. 

## CamelCase JSON
[djangorestframework-camelcase](https://github.com/vbabiy/djangorestframework-camel-case)는 camel case JSON 렌더러와 파서를 지원한다. 