# Responses
REST framework는 콘텐츠를 리턴하는 `Response` 클래스로 HTTP content negotiation을 지원한다. `Response`는 클라이언트 요청에 따라 여러 콘텐츠 타입으로 렌더링할 수 있다.   

`Response`클래스는 Django의 `SimpleTemplateResponse`의 하위 클래스이다. `Response` 객체는 데이터로 초기화된다. 이때 데이터는 원시 Python 기본 요소로 구성되어야한다. 그후 REST framework는 표준 HTTP content negotiation을 사용해 최종 응답 내용을 렌더링하는 방법을 결정한다. 

`Response`클래스를 사용할 필요는 없고 필요한 경우 일반 `HttpResponse`또는 `StreamingHttpResponse` 객체를 뷰에서 리턴할 수도 있다. `Response`클래스를 사용하면 여러가지 형식으로 렌더링할 수 있는 content-negotiationed Web API 응답을 리턴하기에 더 좋은 인터페이스가 제공된다. 

REST framework를 많이 커스터마이징하지 않으려면, `Response`객체를 리턴하는 view에 항상 `APIView`클래스 또는 `@api_view` 함수를 사용해야한다. 이렇게 하면 view에서 content negotiation을 수행하고 응답에 적합한 renderer를 선택하여 view에서 리턴할 수 있다.

## Creating Responses
### Response()
**Signature:** `Response(data, status=None, template_name=None, headers=None, content_type=None)`

일반적인 `HttpResponse`객체와 달리 렌더링된 콘텐츠로 `Response`객체를 인스턴스화하지 않는다. 대신 Python primitives로 구성된 렌더링되지 않은 데이터를 전달한다.

`Response`클래스에서 사용하는 renderer는 Django 모델 인스턴스와 같은 복잡한 데이터 유형을 처리할 수 없으므로 `Response`객체를 만들기 전에 데이터를 원시 데이터 유형으로 직렬화해야한다. 

REST framework의 `Serializer`클래스를 사용해 데이터 serialize하거나 custom serializer를 사용할 수 있다. 

Arguments:

- `data`: response에 대한 직렬화된 데이터
- `status`: response의 status code. default = 200. 자세한 내용은 [status codes](http://www.django-rest-framework.org/api-guide/status-codes/)참고
- `template_name`: `HTMLRenderer`가 선택된 경우 사용할 템플릿 이름
- `headers`: response에 사용할 HTTP header dictionary
- `content_type`: response의 content type. 일반적으로 content negotiation에 따라 렌더러에서 자동으로 설정되지만, conctent type을 명식적으로 지정해야하는 경우가 있다. 

## Attributes
### .data
`Request` 객체의 렌더링 되지 않은 content
### .satus_code
HTTP response의 숫자 상태코드
### .content
response의 렌더링된 content. `.contet`에 접근하려면 먼저 `.render()`메서드를 호출해야한다.

### .template_name
`template_name`(제공된 경우)
`HTMLRenderer`또는 다른 커스텀 템플릿 렌더러가 응답에 대해 허용된 렌더러인 경우에만 필요하다. 

### .accepted_renderer
응답을 렌더하는데 사용되는 renderer 인스턴스

### .accepted_media_type
content negotiation 단계에서 선택한 미디어 타입

### .renderer_context
렌더러의 `.render()` 메소드에 전달 될 추가 컨텍스트 정보의 dictionary

view에서 응답이 리턴되기 직전에 `APIView`또는 `@api_view`에 의해 자동으로 설정된다. 
## Standard HttpResponse attributes
`Response`클래스는 `SimpleTemplateResponse`를 확장하고 모든 일반적인 속성과 메서드를 응답에서도 사용할 수 있다.

예를들어 표준 방식으로 응답에 헤더를 설정할 수 있다.  

```python
response = Response()
response['Cache-Control'] = 'no-cache'
```

### .render()

**Signature:**`.render()`

다른 `TemplateResponse`와 마찬가지로 이 메서드는 응답의 직렬화된 데이터를 최종 응답 콘텐츠로 렌더링하기 위해 호출된다. `.render()`가 호출되면 accept_renderer 인스턴스에서 `.render(data, accepted_media_type, renderer_context)`메서드를 호출한 결과로 응답 내용이 설정된다. 

일반적으로 장고의 표준 응답주기에 의해 처리되기 떄문에 `.render()`를 직접 호출할 필요는 없다. 