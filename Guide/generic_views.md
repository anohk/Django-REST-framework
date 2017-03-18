# Generic views
cbv의 주요한 이점 중 하나는 재사용이 가능한 동작을 구성하는 것이다. REST framework는 일반적으로 사용되는 패턴을 제공하는 미리 빌드된 view들을 제공함으로써 이를 활용한다. 

REST framework에서 제공하는 generic view를 사용하면 데이터베이스 모델과 밀접하게 매핑되는 API view를 빠르게 빌드할 수 있다. 

generic view가 API의 요구사항에 맞지 않으면 일반 `APIView`클래스를 사용해 dropdown하거나 generic view에서 사용하는 mixin 및 기본 클래스를 재사용하여 재사용 가능한 generic views set을 작성할 수 있다. 

```python
# example

from django.contrib.auth.models import User
from myapp.serializers import UserSerializer
from rest_framework import generics
from rest_framework.permissions import IsAdminUser

class UserList(generics.ListCreateAPIView):
	queryset = User.objects.all()
	serializer_class = UserSerializer
	permission_classes = (IsAdminUser,)
```

보다 복잡한 경우에는 view 클래스의 다양한 메서드를 재정의할 수도 있다. 

```python
# example
class UserList(generics.ListCreateAPIView):
	queryset = User.objects.all()
	serializer_class = UserSerializer
	permission_classes = (IsAdminUser,)
	
	def list(self, request):
	# Note the user of 'get_queryset()' instead of 'self.queryset'
	queryset = self.get_queryset()
	serializer = UserSerializer(queryset, many=True)
	return Response(serializer.data)
```

간단한 경우에는 `.as_view()`메서드를 사용해 클래스 속성을 전달할 수 있다. 예를들어 URLconf에 다음과 같이 작성할 수 있다. 

```python
# example

url(r'^/users/', ListCreateAPIView.as_view(queryset=User.objects.all(), serializer_class=UserSerializer), name='user-list')
```

# API Reference
## GenericAPIView
Rest framework의 `APIView`를 확장하여 표준 list와 detail view에 일반적으로 필요한 동작을 추가한다.

제공된 각각의 구체적인 generic view는 `GenericAPIView`를 하나 이상의 mixin 클래스와 결합해 빌드된다. 

### Attributes
#### Basic settings:
다음 속성들은 기본 view 동작을 제어한다.

- `queryset`  
	이 view에서 객체를 리턴하는데 사용해야하는 queryset이다. 일반적으로 이 속성을 설정하거나 `get_queryset()`메서드를 오버라이드해야한다. 만약 view 메서드를 오버라이드한다면, 이 속성에 직접 접근하는 것 대신 `get_queryset()`을 호출해야한다. `queryset`은 한 번 평가되고 그 결과는 모든 후속 요청에 대해 캐시된다. 
- `serializer_class`  
	입력의 검증과 직렬화 복원 및 출력 직력화에 사용하는 serialize class이다. 일반적으로 이 속성을 설정하거나 `get_serializer_class()`메서드를 오버라이드해야한다.
- `lookip_field`  
	개별 모델 인스턴스의 객체 조회를 수행하는데 사용하는 모델 필드이다. 기본값은 `pk`이다. 하이퍼링크된 API를 사용할 때 커스텀 값을 사용해야하는 경우, API view 및 serializer 클래스가 조회 필드를 설정해야한다. 
- `lookup_url_kwarg`  
	객체 검색에 사용하는 URL 키워드 argument이다. URL_conf에는 이 값에 해당하는 키워드 argument가 포함되어야한다. 설정을 해제하면 `lookup_field`와 동일한 값을 사용한다. 

#### Pagination:
다음 속성은 list view와 함께 사용될 때 페이지네이션을 제어하는데 사용된다. 

- `pagination_class`  
	list 결과를 페이지네이션 해야하는 경우 사용하는 pagination 클래스이다. 기본값은 setting에 지정된 `DEFAULT_PAGINATION_CLASS`의 설정값(`rest_framework.pagination.PageNumberPagination`)과 동일하다. `pagination_class=None`으로 설정하면 view에서 페이지네이션을 사용할 수 없다. 

#### Filtering:

- `filter_backends`  
	쿼리셋을 필터링하는데에 사용하는 필터 백엔드 클래스의 목록이다. 기본값은 setting에 설정된 `DEFAULT_FILTER_BACKENDS`의 설정값과 동일하다. 
	
### Methods
#### Base methods:

**`get_queryset(self)`**  
list view에 사용되는 쿼리셋을 리턴한다. detail view내의 lookup 베이스로 사용된다. `queryset` 속성에 의해 지정된 queryset을 리턴하는 것이 디폴트이다. 

이 메서드는 항상 `self.queryset`에 직접 접근하는 것 대신 사용해야하며 `self.queyset`은 한 번만 평가되고 그 결과는 모든 후속 요청에 대해 캐시된다. 

요청을 작성하는 사용자에게 지정된 쿼리 집합 리턴과 같은 동적 동작을 제공하도록 오버라이드할 수 있다. 

```python
# example

def get_queryset(self):
	user = self.request.user
	return user.accounts.all()
```

**`get_object(self)`**  
detail view에 사용해야하는 객체 인스턴스를 리턴한다. 기본적으로 `lookup_field` 매개변수를 사용해 기본 쿼리셋을 필터링한다. 

둘 이상의 URL kwrag를 기반으로 하는 객체 조회처럼 복잡한 동작을 제공하기 위해 무시할 수 있다. 

```python
# example

def get_object(self):
	queryset = self.get_queryset()
	filter = {}
	for field in self.multiple_lookup_fields:
	filter[field] = self.kwargs[field]
	
	obj = get_object_or_404(queryset, **filter)
	self.check_object_permission(self.request, obj)
	return obj
```

API에 객체 수준의 권한이 없으면 선택적으로 `self.check_object_permissions`를 제외하고 단순히 `get_object_or_404`조회에서 객체를 리턴할 수 있다. 

**`filter_queryset(self, queryset)`**  
주어진 쿼리셋을 사용중인 필터 백엔드를 사용해 새로운 쿼리셋을 리턴한다. 

```python
# example

def filter_queryset(self, queryset):
	filter_backends = (CategoryFilter,)
	
	if 'geo_route' in self.request.query_params:
		filter_backends = (GeoRouteFilter, CategoryFilter)
	elif 'geo_point' in self.request.query_params:
		filter_backends = (GeoPointfilter, CategoryFilter)
		
	for backend in list(filter_backends):
		queryset = backend().filter_queryset(self.request, queryset, view=self)
		
	return queryset
```

**`get_serializer_class(self)`**  
serializer에 사용하는 클래스를 리턴한다. 기본값은 `serializer_class`속성을 리턴하는 것이다. 

읽기 및 쓰기 작업에 다른 serializer를 사용하거나 다른 유형의 사용자에게 다른 serializer를 제공하는 등의 동적 동작을 제공하기 위해 오버라이드 할 수 있다. 

```python
# example
def get_serializer_class(self):
	if self.request.user.is_staff:
		return FullAccountSerializer
	return BasicAccountSerializer
```

#### Save and deletion hooks:
다음의 메소드들은 mixin 클래스에서 제공되며 객체 저장 또는 삭제 동작을 쉽게 대체할 수 있다. 

- `perform_create(self, serializer)`  
	새로운 객체 인스턴스를 저장할 때 `CreateModelMixin`에 의해 호출된다. 
- `perform_update(self, serializer)`  
	새로운 객체 인스턴스를 저장할 때 `UpdateModelMixin`에 의해 호출된다. 
- `perform_destroy(self, serializer)`  
	새로운 객체 인스턴스를 저장할 때 `DestroyModelMixin`에 의해 호출된다. 
	
이러한 hooks는 요청에 포함되어있지만 요청 데이터의 일부가 아닌 속성을 설정하는 것에 특히 유용하다. 예를들어 요청 사용자를 기준으로 또는 URL 키워드 argument를 기반으로 객체의 속성을 설정할 수 있다. 

```python
# example

def perform_create(self, serializer):
	serializer.save(user=self.request.suer)
```

이러한 오버라이드 포인트는 확인 이메일을 보내거나, 업데이트를 로깅하는 것과 같이 객체 저장 전후에 발생하는 동작을 추가 할 때 특히 유용하다. 

```python
# example

def perform_update(self, serializer):
	instance = serializer.save()
	send_email_confirmation(user=self.requst.user, modified=instance)
```	

`ValidationError()`를 발생시켜 추가 유효성 검사를 제공하기위해 이러한 훅을 사용할 수도 있다. 데이터베이스 저장 시점에 적용할 유효성 검증 로직이 필요한 경우 유용하다. 

```python
# example
def perform_create(self, serializer):
	queryset = SignupRequest.objects.filter(user=self.request.user)
	if queryset.exists():
		raise ValidationError('You have already signed up')
	serializer.save(user=self.request.user)
```

#### Other methods:
`GenericAPIView`를 사용해 커스텀 view를 작성해야하는 경우 호출해야할 수도 있지만, 일반적으로 다음 메서드를 대체할 필요는 없다. 

- `get_serializer_context(self)`  
	serializer에 제공되는 추가 컨텍스트가 포함된 dictionary를 리턴한다. 기본값은 `request`, `view`및 `format`키를 포함한다.
- `get_serializer(self, instance=None, data=None, many=False, partial=False)`  
	serializer 인스턴스를 리턴한다.
- `get_paginated_response(self, data)`  
	`Response`객체의 페이지네이션 스타일을 리턴한다.
- `paginate_queryset(self, queryset)`  
	필요한 경우 쿼리셋을 페이지네이션하고 페이지 객체를 리턴한다. view에 페이지네이션이 구성되지 않은 경우 None을 리턴한다.   
- `filter_queryset(self, queryset)`  
	주어진 쿼리셋을 사용중인 필터 백엔드를 사용해 새로운 쿼리셋을 리턴한다.  

## Mixins
## ListModelMixin
## CreatedModelMixin
## RetrieveModelMixin
## UpdateModelMixin
## DestroyModelMixin

# Concrete View Classes
## CreateAPIView
## ListAPIView
## RetrieveAPIView
## DestroyAPIView
## UpdateAPIView
## ListCreateAPIView
## RetrieveUpdateAPIView
## RetrieveDestroyAPIView
## RetrieveUpdateDestroyAPIView

# Customizing the generic views
## Creating custom mixins
## Creating custom base classes

# PUT as create

-

# Third party packages
## Django REST Framework bulk
## Django Rest Multiple Models
