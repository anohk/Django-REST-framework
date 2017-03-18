#Requests
Rest framework의 `Request`클래스는 표준 `HttpRequest`를 확장하여 Rest framework의 유연한 요청 구문 분석 및 요청 인증을 지원한다.

##Request parsing
REST framework의 Request 객체는 유연한 요청 구문 분석 기능을 제공하므로, 사용자가 일반적으로 form data를 다루는 것과 같은 방법으로 JSON 데이터 혹은 다른 media type으로 요청을 처리할 수 있다. 

###.data
`request.data`는 request body의 내용을 분석해서 리턴한다. 이는 아래의 내용을 제외하고는 표준 `request.Post`와 `request.FILES`속성과 유사하다. 

- file과 non-file input을 포함해 파싱된 모든 내용을 포함한다.
- `POST`외의 다른 HTTP method의 내용을 파싱하는 것을 지원한다. `PUT`이나 `PATCH`와 같은 요청 내용에 접근할 수 있다.
- form data 보다 유연한 파싱을 지원한다. 

###.query_params
`request.query_params`는 `request.GET`을 더 명확하게 명명하는 것이다. 

코드를 명시적으로 작성하기 위해, Django의 표준 `request.GET`을 사용하는 대신 `request.query_params`를 사용하는 것을 추천한다. 

###.parsers
`APIView`클래스 또는 `@api_view`데코레이터는 view에 설정된 `parser_classes`또는 settings에 설정된`DEFAULT_PARSER_CLASSES`에 따라 자동으로 속성이 `Parser` 인스턴스 리스트로 설정되도록한다.

##Content negotiation
###.accepted_render
###.accepted\_media\_type

##Authentication
REST framework는 다음과 같이 유연하게 요청 별 인증을 제공한다. 

- API의 다른 부분에 대해 서로 다른 인증 정책을 사용
- 다중 인증 정책의 사용을 지원
- 들어오는 요청과 관련된 사용자 및 토큰 정보를 제공

###.user
`request.user`는 일반적으로 `django.contrib.auth.models.User`의 인스턴스를 리턴하지만 동작은 사용되는 인증 정책에 따라 다르다.

요청이 인증되지 않은 경우 `request.user`의 기본값은 `django.contrib.auth.models.AnonymousUser`의 인스턴스이다.

더 자세한 내용은 [authentication documentaion](http://www.django-rest-framework.org/api-guide/authentication/) 참고

###.auth
`request.auth`는 추가적인 인증 컨텍스트를 리턴한다. `request.auth`의 정확한 동작은 사용되는 인증 정책에 따라 다르지만, 대개 요청이 인증된 토큰의 인스턴스일 수 있다.

만약 요청이 인증되지 않았거나 추가적인 인증 컨텍스트가 없는 경우 `request.auth`의 기본값은 `None`이다.

더 자세한 내용은 [authentication documentaion](http://www.django-rest-framework.org/api-guide/authentication/) 참고

###.authenticators
`APIView`클래스 또는 `@api_view`데코레이터는 view에 설정된 `authentication_calsses`또는 settings에 설정된`DEFAULT_AUTHENTICATORS`에 따라 자동으로 속성이 `Athentication` 인스턴스 리스트로 설정되도록한다.

##Browser enhancements
REST framework는 브라우저 기반 PUT, PATCH 및 DELETE form과 같은 브라우저 개선 사항을 지원한다.
###.method
`request.method`는 요청의 HTTP 메서드의 대문자로 된 문자열 표현을 리턴한다.  
브라우저 기반 PUT, PATCH, DELETE form이 지원된다. 

자세한 내용은 [browser enhancements documnetation](http://www.django-rest-framework.org/topics/browser-enhancements/) 참고

###.content_type
`request.content_type`은 HTTP 요청의 본문 미디어 유형을 나타내는 문자열 객체를 리턴한다. 미디어 유형이 제공되지 않은 경우에는 빈 문자열을 리턴한다.

일반적으로 REST framework의 기본 요청 구문 분석 동작에 의존하므로 요청의 콘텐츠 타입에 직접 접근 할 필요는 없다. 

콘텐츠 타입에 접근해야 하는 경우, 브라우저 기반 비 형식 콘텐츠에 대한 투명한 지원을 제공하므로 `request.META.get('HTTP_CONTENT_TYPE')`보다 `.content_type`속성을 사용해야한다. 

자세한 내용은 [browser enhancements documnetation](http://www.django-rest-framework.org/topics/browser-enhancements/) 참고

###.stream
`request.stream`은 요청 본문의 내용을 나타내는 stream을 리턴한다. 

일반적으로 REST framework의 기본 요청 구문 분석 동작에 의존하므로 요청의 콘텐츠에 직접 접근 할 필요는 없다. 

##Standard HttpRequest attributes
REST framework의 `Request`는 Django의 `HttpRequest`를 확장했기 때문에, 다른 표준 속성과 메서드를 모두 사용할 수 있다. 예를들어 `request.META`와 `request.session` dictionary를 이용할 수 있다. 

구현을 위해 `Request` 클래스는 `HttpRequest`클래스에서 상속하지 않고 composition을 사용해 클래스를 확장한다. 


