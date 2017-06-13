# Class-based Views

REST framework 는 Django의 `View`클래스의 서브클래스인 `APIView`클래스를 제공한다. 

`APIView`클래스는 아래와 같이 일반 `View`클래스와 다르다. 

- 핸들러 메서드에 전달된 요청은 Django의 HttpRequest 인스턴스가 아닌 REST framework의 `Request` 인스턴스가 된다.
- 핸들러 메서드는 Django의 `HttpResponse`대신 REST framework의 `Response`를 리턴할 수 있다. view는 content negotiation을 관리하고 응답에서 올바른 렌더러를 설정한다
- 모든 `APIException` 예외는 적절한 응답으로 조정된다.
- 들어오는 요청은 인증되고, 적절한 권한 또는 throttle 체크로 요청을 핸들러 메서드로 전달하기 전에 실행된다.

> throttle? 

`APIView`클래스를 사용하는 것은 일반 View 클래스를 사용하는 것과 거의 동일하다. 들어오는 요청은 `.get()`또는 `.post()`와 같은 적절한 핸들러 메서드로 전달된다. 또한 API 정책의 다양한 측면을 제어하는 여러가지 속성을 클래스에 설정할 수 있다. 

```python
# example

from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import authentication, permissions

class ListUsers(APIView):
	"""
	View to list all users in the system.
	
	* Requires token authentication.
	* Only admin users are able to access this view.
	"""

	authentication_classes = (authentication.TokenAuthentication,)
	permission_classes = (permissions.IsAdminUser,)
	
	def get(self, request, format=None)
	"""
	Return a list of all users.
	"""
	username = [user.username for user in User.objects.all()]
	return Response(usernames)
```

## API policy attributes
API views의 플러그 가능한 부분을 제어하는 속성들
### .renderer_classes
### .parser_classes
### .authentication_classes
### .throttle_classes
### .permission_classes
### .content_negotiation_class

## API policy instantiation methods
다음 메서드들은 REST framework에서 다양한 플러그 가능한 API 정책을 인스턴스화하는데 사용된다. 일반적으로 이런 메서드를 재정의할 필요는 없다. 
### .get_renderers(self)
### .get_parsers(self)
### .get_authenticators(self)
### .get_throttles(self)
### .get_permissions(self)
### .get_content_negotiator(self)
### .get_exception_handler(self)

## API policy implementation methods
다음 메서드들은 핸들러 메서드에 전달하기 전에 호출된다. 
### .check_permissions(self, request)
### .check_throttles(self, request)
### .perform_content_negotiation(self, request, force=False)

## Dispatch methods
다음 메서드들은 view의 `.dispatch()`메서드에 의해 직접 호출된다. 이들은 `.get()`, `.post()`, `.put()`, `.patch()`, `.delete()`와 같은 핸들러 메서드를 호출하기 전후에 수행해야하는 모든 것을 수행한다. 

### .initial(self, request, \*args, **kwargs)
사용 권한 및 제한을 적용하고 content negotiation을 수행하는데 사용된다. 

### .handle_exception(self, exc)
핸들러 메서드에 의해 throw 된 예외는 이 메서드로 전달된다. `Response` 인스턴스를 리턴하거나 예외를 다시 발생시킨다.

기본 구현에서는 Django의 `Http404` 및 `PermissionDenied` 예외 뿐 아니라 `rest_framework.exceptions.APIException`의 하위 클래스를 처리하고 적절한 오류 응답을 반환한다. 

API에서 리턴하는 오류 응답을 커스터마이징 해야하는 경우 이 메서드를 서브클래스화 해야한다. 

### .initialize_request(self,request, \*args, **kwargs)
핸들러 메서드에 전달된 request 객체가 일반적인 Django `HttpRequest`가 아닌 `Request`의 인스턴스인지 확인한다. 

### .finlize_response(self,request, response, \*args, **kwargs)
핸들러 메서드에서 리턴된 모든 `Response` 객체가 content-negotiation에 의해 결정된 대로 올바른 내용의 타입으로 렌더링되도록 한다. 

# Fuction Based Views
REST framework를 사용하면 일반 function based view로 작업할 수 있다. `Request`의 인스턴스를 받을 수 있도록 fbv를 래핑하는 간단한 데코레이터 집합을 제공한다. `Response`를 리턴하고 요청을 처리하는 방법을 구성할 수 있다. 

## @api_view()
**Signature:**`@api_view(http_method_names=['GET'], exclude_from_schema=False)`

이 기능의 핵심은 view가 응답해야하는 HTTP 메서드의 목록을 취하는 `api_view`데코레이터이다. 예를들면, 다음은 몇 가지 데이터를 수동으로 반환하는 간다난 view를 작성하는 방법이다. 

```python
from rest_framework.dcorators import api_view

@api_view()
def hello_world(request):
	return Response({"message": "Hello, world!"})
```
이 view는 설정에 지정된 기본 renderer, parser, authentication class등을 사용한다.

기본적으로 `GET`메서드만 허용된다. 다른 방법은 '405 Method Not Allowed'로 응답한다. 이 동작을 변경하려면 view에서 지정해준다. 

```python
@api_view(['GET', 'POST'])
def hello_world(request):
	if request.method == 'POST':
		return Response({"messgae": "Got some data", 
		"data": request.data})
	return Response({"message": "Hello, world!"})
```
`exclude_from_schema` 인수를 사용해 API view를 자동 생성 스키마에서 생략된 것으로 표시할 수도 있다. 

```python
@api_vew(['GET'], exclude_from_schema=True)
def api_docs(request_:
	...
```

## API policy decorators
기본 설정을 재정의하기 위해 REST framework는 view에 추가할 수 있는 추가 데코레이터를 제공한다. 
이것들은 `@api_view`데코레이터 다음에 와야한다.
예를들어 [throttle](http://www.django-rest-framework.org/api-guide/throttling/)을 사용하여 특정 사용자가 하루에 한번만 호출할 수 있도록 view를 만드려면 `@throttle_classes`데코레이터를 사용해 throttle class list를 전달한다. 

```python
from rest_framework.decorators import api_view, throttle_classes
from rest_framework.throttling import UserRateThrottle

class OncePerDayUserThrottle(UserRateThrottle):
	rate = '1/day'
	
@api_view(['GET'])
@throttle_classes([OncePerDayUserThrottle])
def view(request):
	return Response({"message": "Hello for today! See you tommorow!"})
```

이런 데코레이터는 위에서 설명한 `APIView`하위 클래스에 설정된 속성에 해당한다. 

사용가능한 데코레이터들:

- `@renderer_classes(...)`
- `@parser_classes(...)`
- `@authentication_classes(...)`
- `@throttle_classes(...)`
- `@permission_classes(...)`

이러한 데코레이터 각각은 클래스의 리스트 또는 튜플인 단일 argument를 취한다.