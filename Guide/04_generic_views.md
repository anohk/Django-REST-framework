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
mixin 클래스는 기본 view 동작을 제공하는데 사용하는 액션들을 제공한다. mixin 클래스는 `.get()`, `.post()`와 같은 핸들러 메서드를 직접 정의하는 것이 아니라 작업 메서드를 제공하는 것이다. 이는 더욱 유연한 행동 구성을 가능하게 한다. 

mixin 클래스는 `rest_framework.mixins`에서 임포트 할 수 있다. 

## ListModelMixin
쿼리셋을 리스트로 구현하는 `.list(request, *args, **kwargs)` 메서드를 제공한다. 
queryset이 채워지면, 응답의 본문으로 queryset의 직렬화된 표현과 함께 `200 OK` 응답을 리턴한다. 응답 데이터는 선택적으로 페이징될 수 있다. 

## CreatedModelMixin
새로운 모델 인스턴스의 생성 및 저장을 구현하는 `.create(request, *args, **kwargs)` 메서드를 제공한다. 

## RetrieveModelMixin
응답에서 기존 모델 인스턴스를 리턴하도록 구현하는 `.retrieve(request, *args, **kwargs)` 메서드를 제공한다. 

객체를 검색할 수 있는 경우 `200 OK`응답을 리턴하고 객체를 응답 본문으로 직렬화해서 리턴한다. 그렇지 않으면 `404 Not Found`를 리턴한다. 

객체가 생성되면 객체의 직렬화된 표현을 응답의 본문으로 사용하고 `201 Created` 응답을 리턴한다. 표현에 `url`이라는 키가 포함되어 있으면 응답의 `Location` 헤더가 해당 값으로 채워진다.

객체 생성을 위해 제공된 요청 데이터가 유요하지 않은 경우 `400 Bad Request`가 리턴되고 오류 내용은 응답 본문으로 리턴된다. 

## UpdateModelMixin
기존 모델 인스턴스를 업데이트하고 저장하는 `.update(request, *args, **kwargs)`메서드를 제공한다. 

또한 `update` 메서드와 유사한 `.partial_update(request, *args, **kwargs)` 메서드를 제공한다. HTTP `PATCH` 요청을 지원하며 업데이트의 모든 필드는 선택사항이다. 

객체가 업데이트되면 객체의 직렬화된 표현이 응답 본문과 함께 `200 OK`응답을 리턴한다. 

객체를 업데이트하기 위해 제공된 요청 데이터가 유효하지 않은 경우 `400 Bad Request`가 리턴되고, 오류의 세부 정보가 응답 본문으로 사용된다. 

## DestroyModelMixin
기존 모델 인스턴스의 삭제를 구현하는 `.destroy(request, *args. **kwargs)` 메서드를 제공한다. 

객체가 삭제되면 `204 No Content`를 리턴하고 그렇지 않으면 `404 Not Found`를 리턴한다. 

# Concrete View Classes
아래의 클래스들은 구체적인 generic view이다. generic view를 사용한다면, 많은 커스텀 동작이 필요하지 않을 때 적용할 만한 레벨이다. 

view 클래스는 `rest_framework.generics`에서 임포트 할 수 있다. 

## CreateAPIView
**create-only** 엔드포인트에 사용된다.  

`post` 메서드 핸들러를 제공한다.  

Extends: `GenericAPIView`, `CreateModelMixin`

## ListAPIView
**read-only** 엔드포인트가 모델 인스턴스 컬렉션을 나타내는데 사용된다.  

`get` 메서드 핸들러를 제공한다.  

Extends: `GenericAPIView`, `ListModelMixin`

## RetrieveAPIView
**read-only** 엔드포인트가 단일 모델 인스턴스를 나타내는데 사용된다.  

`get` 메서드 핸들러를 제공한다.  

Extends: `GenericAPIView`, `RetrieveModelMixin`

## DestroyAPIView
단일 모델 인스턴스의 **delete-only** 엔드포인트에 사용된다. 

`delete` 메서드 핸들러를 제공한다.

Extends: `GenericAPIView`, `DestroyModelMixin`

## UpdateAPIView
단일 모델 인스턴스의 **update-only** 엔드포인트에 사용된다. 

`put`, `patch` 메서드 핸들러를제공한다.

Extends: `GenericAPIView`, `UpdateModelMixin`

## ListCreateAPIView
모델 인스턴스 콜렉션을 나타내는 **read-write** 엔드포인트에 사용된다. 

`get`, `post` 핸들러를 제공한다. 

Extends: `GenericAPIView`, `ListModelMixin`, `CreateModelMixin`

## RetrieveUpdateAPIView
단일 모델 인스턴스를 나타내기위해 **read or update** 엔드포인트에 사용된다.

`get`, `put`, `patch` 핸들러를 제공한다. 

Extends: `GenericAPIView`, `RetrieveModelMixin`, `UpdateModelMixin`

## RetrieveDestroyAPIView
단일 모델 인스턴스를 나타내는 `read or delete` 엔드포인트에 사용된다. 

`get`, `delete` 핸들러를 제공한다. 

Extends: `GenericAPIView`, `RetrieveModelMixin`, `DestroyModelMixin`

## RetrieveUpdateDestroyAPIView
단일 모델 인스턴스를 나타내는 `read-write-delte` 엔드포인트에 사용된다.

`get`, `put`, `patch`, `delete` 핸들러를 제공한다. 

Extends: `GenericAPIView`, `RetrieveModelMixin`, `UpdateModelMixin`, `DestroyModelMixin`

# Customizing the generic views
여러 위치에서 커스터마이징한 동작을 재사용하는 경우, 동작을 공통 클래스로 리팩토링하여 필요할 때 모든 view나 viewset에 적용할 수 있다. 

## Creating custom mixins
예를 들어, URL conf 내의 복수의 필드에 의해 객체를 검색해야 할 경우, 다음과 같이 mixin 클래스를 작성할 수 있다. 

```python
class MultipleFieldLookupMixin(object):
	def get_object(self):
		queryset = self.get_queryset()
		queryset = self.filter_queryset(queryset)
		filter = {}
		for field in self.lookup_fileds:
			filter[field] = self.kwargs[field]
		return get_object_or_404(queryset, **filter)
```

그리고나서 커스텀된 동작을 적용할 때 마다 view나 viewset에 적용할 수 있다. 

```python
class RetrieveUserView(MultipleFieldLookupMixin, generics.RetrieveAPIView):
	queryset = User.objects.all()
	serializer_class = UserSerializer
	lookup_fields = ('account', 'username')
```


## Creating custom base classes
여러 view에서 mixin을 사용하는 경우, 프로젝트 전체에서 사용할 수 있는 기본 view의 set을 만들 수 있다. 

```python
class BaseRetrieveView(MultipleFieldLookupMixin,
							generics.RetrieveAPIView):
	pass

class BaseRetrieveUpdateDestroyView(MultipleFieldLookupMixin,
											generics.RetrieveUpdateDestroyAPIView):
	pass					
```

프로젝트 전반에 걸쳐 여러 view에서 동일하게 반복해야하는 커스텀 동작이 있는 경우, custom base class 를 사용하는 것이좋다. 