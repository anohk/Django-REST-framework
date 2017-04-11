# Authentication

Authentication은 요청을 보낸 사용자 또는 서명된 토큰과 같은 식별 자격 증명 세트와 연결하는 매커니즘이다. 권한 및 제한 정책은 이런 자격 증명을 사용해 요청을 허용해야하는지 결정할 수 있다. 

REST framework는 여러 가지 인증 스키마를 제공하며, 커스텀 스키마를 구현할 수도 있다. 

인증은 항상 view의 맨 처음에 실행된다. 권한 및 제한 검사가 수행되기 전에, 다른코드가 진행되기 전에 실행됨.

`request.user` 속성은 일반적으로 `contrib.auth`패키지의 `User` 클래스 인스턴스로 설정된다. 

`request.auth` 등록 정보는 추가 인증 정보에 사용된다. 예를들면 요청이 서명된 인증 토큰을 나타내는데에 사용될 수 있다. 

**NOTE**  
인증 자체만으로는 들어오는 요청을 허용하거나 거부하지 않으며, 단순히 요청이 만들어진 자격 증명을 식별한다.

## How authentication is determined

인증 체계는 항상 클래스 목록으로 정의된다. REST framework는 각 목록의 클래스에 대해 인증을 시도하고, 성공적으로 인증한 첫 번째 클래스의 리턴 값을 사용해 `request.user`와 `request.auth`를 설정한다. 

만약 인증된 클래스가 없으면, `request.user`는 `django.contrib.auth.models.AnonymousUser`의 인스턴스로 설정되고, `request.auth`는 `None`으로 설정된다. 

인증되지 않은 요청에 대한 `request.user`와 `request.auth`의 값은 `UNAUTHENTICATED_USER`와 `UNAUTHENTICATED_TOKEN`설정을 사용해 수정할 수 있다. 

## Setting the authentication scheme
`DEFAULT_AUTHENTICATION_CLASSES` 설정을 사용해 기본 인증 체계를 전역으로 설정할 수 있다.

```python
REST_FRAMEWORK = {
	'DEFAULT_AUTHENTICATION_CLASSES':(
	'rest_framework.authentication.BasicAuthentication',
	'rest_framework.authentication.SessionAuthentication',
	)
}
```
또한 `APIView` 클래스 베이스 view를 사용해 view 단위별 인증 체계를 설정할 수 있다. 

```python
from rest_framework.authentication import SessionAuthentication, BasicAuthentciation
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
	authentication_classes = (SessionAuthentication, BasicAuthentication)
	permission_classes = (IsAuthenticated,)
	
	def get(self, request, format=None):
		content = {
			'user': unicode(request.user),
			'auth': unicode(request.auth),
		}
```
또는 `@api_view` 데코레이터를 사용한 함수 베이스 view에서 사용할 수 있다. 

```python
@api_view(['GET'])
@authentication_classes((SessionAuthentication, BasicAuthentication))
@permissions_classes((IsAuthenticated,))
def example_view(request, format=None):
	content = {
		'user': unicode(request.user),
		'auth': unicode(request.auth),
	}
	return Response(content)
```

## Unauthorized and Forbidden responses
인증되지 않은 요청에 권한이 거부될 때 적절한 두 개의 오류코드가 있다. 

- HTTP 401 Unauthorized
- HTTP 403 Permission Denied

HTTP 401 응답에는 항상 클라이언트에 인증 방법을 지시하는 `WWW-Authenticate` 헤더가 포함되어야 한다.   
HTTP 403 응답에는 `WWW-Authenticate` 헤더가 포함되지 않는다. 

인증 체계에 따라 사용되는 응답의 종류는 달라진다.   
응답 유형을 판별하기 위해 여러 인증 체계가 사용중이더라도 하나의 scheme 만 사용될 수 있다. **view 에 설정된 첫 번째 인증 클래스는 응답 유형을 결정할 때 사용된다.**   

요청이 성공적으로 인증되었지만, 요청을 수행할 수 있는 권한이 거부된 경우에는, 인증 구성에 관계없이 `403 Permission Denied` 응답이 사용된다. 

# API Reference
## BasicAuthentication
이 인증 체계는 사용자의 username과 password에 대해 서명된 HTTP 기본 인증을 사용한다. 기본 인증은 일반적으로 테스트에만 적합하다. 

성공적으로 인증되면 `BasicAuthentication`은 다음의 자격 증명을 제공한다. 

- `request.user`: Django의 `User` 인스턴스
- `request.auth`: `None`

권한이 거부된 인증되지않은 응답은 적절한 WWW-Authenticate 헤더와 함께 `HTTP 401 Unauthorized` 응답을 한다. 

`WWW-Authenticate: Basic realm='api`

**NOTE**  
프로덕션 환경에서 `BasicAuthentication`을 사용하는 경우 `https`를 통해서만 API를 사용해야한다. 또한 API 클라이언트가 로그인 할 때 항상 username과 password를 재요청하고, 해당 세부 정보를 영구 저장소에 저장하지 않아야한다. 

 
## TokenAuthentication
이 인증 체계는 간단한 토큰 기반 HTTP 인증 체계를 사용한다. 
토큰 인증은 네이티브 데스크톱 및 모바일 클라이언트와 같은 클라이언트-서버 설정에 적합하다. 

`TokenAuthentication` 스킴을 이용하는 경우, `TokenAuthentication`을 포함할 수 있도록 인증 클래스를 구성하고, `INSTALLED_APPS` 설정에 `rest_framework.auttoken`을 추가해야한다. 

```python
INSTALLED_APPS = (
	...
	'rest_framework.authtoken'
)
```

**NOTE**  
설정을 변경 한 후 `manage.py migrate`를 실행해야한다. `rest_framework.authtoken` 앱은 Django 데이터베이스 마이그레이션을 제공한다. 

또한 사용자를 위한 토큰을 만들어야한다. 

```python
from rest_framework.authtoken.models import Token

token = Token.objects.create(user=..)
print token.key
```

클라이언트가 인증할 수 있도록, 토큰 키는 `Authorization` HTTP 헤더에 포함되어야한다.   
키에는 공백으로 구분하여 문자열 `Token`을 접두어로 사용해야한다. 

`Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
`

**NOTE**  
`bearer`와 같이 헤더에 다른 키워드를 사용하고 싶으면 `TokenAuthentication`을 하위 클래스로 만들고 키워드 클래스 변수를 설정해야한다. 

성공적으로 인증되면 `TokenAuthentication`은 다음과 같은 자격 증명을 제공한다. 

- `request.user`: Django의 `User` 인스턴스
- `request.author`: `rest_framework.authtoken.models.Token`의 인스턴스

권한이 거부된 인증되지 않은 응답은 적절한 WWW-Authenticate 헤더와 함께 `HTTP 401 Unauthorized` 응답이 된다. 

`WWW-Authenticate: Token`

`curl` 커맨드 라인 도구는 토큰으로 인증된 API를 테스트하는 데에 유용할 수 있다.   
`curl -X GET http://127.0.1:8000/api/example/ -H 'Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b'`

**NOTE**  
프로덕션 환경에서 `TokenAuthentication`을 사용하는 경우 `https`를 통해서만 API를 사용해야한다. 

### Generating Tokens
#### By Using signals
모든 사용자가 자동으로 생성된 토큰을 갖기 원한다면 사용자의 `post_save` 신호를 간단하게 잡을 수 있다. 

```python
from django.conf import settings
from django.db.models.signals import post_save
from django.dispatch import receiver
from rest_framework.authtoken.models import Token

@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_auth_token(sender, instance=None, created=False, **kwargs):
	if created:
		Token.objects.create(user=instance)
```

이 코드 조각은 `models.py` 모듈이나 시작시 Django가 가져올 다른 위치에 배치해야한다. 

이미 사용자가 생성된 경우, 다음과 같이 기존 사용자에 대한 토큰을 생성할 수 있다. 

```python
from django.contrib.auth.models import User
from rest_framework.authtoken.models import Token

for user in User.objects.all():
	Token.objects.get_or_create(user=user)
```

### By exposing an api endpoint
`TokenAuthentication`을 사용할 때 클라이언트가 username과 password가 있는 토큰을 얻게 하는 매커니즘을 제공할 수 있다. REST framework는 built-in view를 통해 이 동작을 제공한다. 이를 사용하기 위해서, URLconf에 `obtain_auth_token`을 추가해준다.

```python
from rest_framework.authtoken import views
urlpatterns += [
	url(r'^api-token-auth/', views.obtain_auth_token)
]
```

`obtain_auth_token`은 유효한 `username`과 `password`가 form data 또는 JSON을 사용해 POST되면, JSON response를 리턴한다. 

`{'token' : '9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b'}`

기본 `obtain_auth_token` view는 설정에서 기본 renderer 및 parser 클래스를 사용하는 대신, JSON 요청 및 응답을 명시적으로 사용한다. `obtain_auth_token` view를 커스터마이징 하려면 `ObtainAuthToken` view class를 오버라이드하여 URLconf에 사용하면된다. 

기본적으로 `obtain_auth_token` view에 적용된 사용권한이나 제한은 없다. 제한을 적용하려면 view class를 재정의하고, `throttle_classes` 속성을 사용해 view class를 포함해야한다. 

### with Django admin
관리자 인터페이스를 통해 수동으로 토큰을 생성할 수도 있다. 대규모 사용자 기반을 사용하는 경우, 사용자 필드를 `raw_field`로 선언하여 `TokenAdmin`을 패치해 커스터마이즈하는 것이 좋다. 

`app/admin.py`

```python
from rest_framework.authtoken.admin import TokenAdmin

TokenAdmin.raw_id_fields = ('user',)
```