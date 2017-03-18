# VeiwSets

Django REST framework를 사용하면 `ViewSet` 이라는 단일 클래스에서 관련된 view의 set에 대한 로직을 결합할 수 있다. 다른 프레임워크에서는 `Resources` 또는 `Controlloers` 같은 개념적으로 유사한 구현을 찾아 볼 수 있다.  

`ViewSet` 클래스는  **class-based View** 타입으로 `.get()`이나 `.post()`같은 메서드 핸들러를 제공하는 것이 아니라, `.list()`나 `.create()`와 같은 액션을 제공한다. 

`ViewSet`의 메서드 핸들러는 `.as_vew()` 메서드를 사용하여 view를 끝내는 시점의 액션만 바인딩된다. 

일반적으로 urlconf의 viewset에 명시적으로 등록하지 않고, router 클래스로 등록하면 자동으로 urlconf가 결정된다.

예시: 시스템의 모든 사용자를 나열하거나 검색하는데 사용할 수 있는 간단한 viewset을 정의한다.

```python
# example

from dajngo.contrib.auth.models import User
from django.shortcuts import get_object_or_404
from myapps.serializers import UserSerializer
from rest_framework import viewsets
from rest_framework.response import Response

class UserViewSet(viewsets.ViewSet):
	def list(self, request):
		queryset = User.objects.all()
		serializer= UserSeriaizer(queryset, many=True)
		return Response(Serializer.data)
		
	def retrieve(self, request, pk=None):
		queryset = User.objects.all()
		user = get_object_or_404(queryset, pk=pk)
		serializer = UserSerializer(user)
		return Response(serializer.data)
```

필요하다면 viewset을 다음과 같이 두 개의 개별 뷰에 바인딩할 수 있다. 

```python
user_list = UserViewSet.as_view({'get': 'list'})
user_detail = UserViewSet.as_view({'get': 'retrieve'})
```

viewset을 router에 등록하고 urlconf가 자동으로 생성되도록 한다. 

```python
from myapp.views import UserViewSet
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'users', UserViewSet)
urlpatterns = router.urls
```

커스텀된 viewset을 작성하는 것 보다 기본적인 동작을 제공하는 기존의 기본 클래스를 사용하는 것이 좋다. 

```python
class UserViewSet(viewsets.ModelViewSEt):
	serializer_class = UserSerializer
	queryset = User.objects.all()
```

`View` 클래스를 사용하는 것 보다 `ViewSet` 클래스를 사용하는 것에는 두 가지 주요한 이점이 있다. 

- 반복 논리를 하나의 클래스로 결합할 수 있다. 위의 예제와 같이 queryset은 한 번만 지정하면 여러 view에서 사용된다. 
- router를 사용함으로써 더이성 URL conf의 연결을 처리 할 필요가 없다. 

일반적인 view와 URL confs를 사용하는 것은 보다 명확한 제어가 가능하다. ViewSets은 빠르게 시작하고 실행하려는 경우, 혹은 대규모 API가 있고 전체적으로 일관된 URL conf를 적용하려는 경우 유용하다. 

## Marking extra actions for routing

REST framework에 포함된 기본 router는 아래와 같이 표준 set(생성/검색/업데이트/삭제 스타일 작업을 위한)의 경로를 제공한다. 

```python
class UserViewSet(viewsets.ViewSet):
	def list(self, request):
		pass
	def create(self, request):
		pass	
	def retrieve(self, request, pk=None):
		pass
	def partial_update(self, request, pk=None):
		pass
	def destroy(self, request, pk=None):
		pass
```

라우팅해야하는 임시 메서드가 있는 경우, `@detail_route`나 `@list_route` 데코레이터를 사용해 라우팅을 요구한다는 것을 표시할 수 있다. 

`@detail_route` 데코레이터는 URL 패턴에 `pk`를 포함하고 단일 인스턴스가 필요한 메서드이다.   
`@list_route` 데코레이터는 객채 목록에서 작동하는 메서들르 대상으로한다. 

```python
from django.contrib.auth.models import User
from rest_framework import status
from rest_framework import viewsets
from rest_framework.decorators import detail_route, list_route
from rest_framework.response import Response
from myapp.serializers import UserSerializer, PasswordSerializer

class UserViewSet(viesets.ModelViewSet):
	queryset = User.objects.all()
	serializer_class = UserSerializer
	
	@detail_route(methods=['post'])
	def set_password(self, request, pk=None):
		user = self.get_object()
		serializer = PasswordSerializer(data=request.data)
		if serializer.is_valid():
			user.set_password(serializer.data['password'])
			user.save()
			return Response({'status': 'password set'})
		else:
			return Response(serializer.errors,
								status=status.HTTP_400_BAD_REQUEST)
								
	@list_route()
	def recent_users(self, request):
		recent_users = User.objects.all().order('-last_login')
		
		page = self.paginate_queryset(recent_users)
		if page is not Noe:
			serializer = self.get_serializer(page, many=True)
			return self.get_paginated_response(serializer.data)
		
		serializer = self.get_serializer(recent_users, mnay=True_
		return Response(serializer.data)
```

이러한 데코레이터는 기본적으로 `GET`요청을 라우트하지만 `methods` argument를 사용해 다른 HTTP 메서드를 채택할 수도 있다. 

```python
@detail_route(methods=['post', 'delete'])
def unset_password(self, request, pk=None):
	...
```

# API Reference
## ViewSet
`ViewSEt` 클래스는 `APIView`에서 상속받는다. viewset에 대한 API 정책을 제어하기 위해 `permission_classes`, `authentication_classes`와 같은 표준 속성을 사용할 수 있다. 

`ViewSet` 클래슨는 액션의 구현을 제공하지 않는다. `ViewSet` 클래스를사용하려면 클래스를 오버라이드하고 액션 구현을 명시적으로 정의해야한다. 

## GenericViewSet
`GenericViewSet` 클래슨느 `GenericAPIView`를 상속받아 `get_object`,`get_queryset`메서드 및 기타 generic view의 기본 동작의 기본 셋을 제공하지만, 기본적으로 어떠한 액션도 포함하지 않는다.

`GenericViewSet` 클래스를사용하려면 클래스를 재정의하고 필요한 mixin 클래스를 혼합하거나 액션 구현을 명시적으로 정의해야한다. 
 
## ModelViewSet
`ModelViewSet` 클래스는 `GenericAPIView`를 상송받고 다양한 mixin 클래스의 동작을 혼합해 다양한 액션에 대한 구현을 표현한다. 

`ModelViewSet` 클래스에서 제공하는 것은 `.list()`, `.retrieve()`, `.update()`, `.partial_update`, `.destroy()`이다. 

`ModelViewSEt`은 `GenericAPIView`를 확장하기 때문에 일반적으로 `queryset` 및 `serializer_class` 속성을 제공해야한다. 

```python
class AccountViewSet(viewsets.ModelViewSet):
	queryset = Account.objects.all()
	serializer_class = AccountSerializer
	permission_classes = [IsAccountAdminOrReadOnly]
```

`GenericQPIView`가 제공하는 표준 속성이나 메서드를 대체하여 사용할 수 있다. 예를들어 작동해야하는 queryset을 동적으로 결정하는 viewset을 사용하려면 다음과 같이 작성할 수 있다. 

```python
class AccountviewSet(viewsets.ModelViewset):
	serializer_class = AccountSerializer
	permission_class = [IsAccountAdminOrReadOnly]
	
	def get_queryset(self):
		return self.request.user.accounts.all()
```

그러나 `ViewSet`에서 `queryset` 속성을 제거하려면 연결된 라우터가 모델의 base_name을 자동으로 파생시킬 수 없기 떄문에 router registraion의 일부로 `base_name` kwargs를 지정해야한다.

또한 이 클래스는 기본적으로 create/list/retrieve/update/destroy 액션의 전체 set 을 제공하지만 표준 권한 클래스를 사용해 사용가능한 작업을 제한할 수 있다. 

## ReadOnlyModelViewSet
`ReadOnlyModelViewSet`은 `GenericAPIView`를 상속받는다. `ModelViewSet`처럼 다양한 액션에 대한 구현도 포함되지만 `ModelViewSet`과 달리 읽기 전용 액션`.list()`, `.retrieve()`만 제공된다. 

`ModelViewSet`과 같이 일반적으로는 적어도 `queryset`과 `serializer_class` 속성을 제공해야한다. 

```python
class AccountViewSet(viewsets.ReadOnlyModelViewSet):
	queryset = Account.objects.all()
	serializer_class = AccountSerializer
```

`ModelViewSet`과 같이 `GenericAPIView`에서 사용할 수 있는 표준 속성 및 메소드를 대체하여 사용할 수 있다. 