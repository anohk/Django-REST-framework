# Routers
Rails와 같은 일부 웹 프레임워크는 응용 프로그램의 URL을 들어오는 요청을 처리하는 로직에 매핑하는 방법을 자동으로 결정하는 기능을 제공한다. 

REST framework는 Django에 대한 자동 URL 라우팅 지원을 추가하고 view 로직을 URL set 에 간단하고 빠르고 일관되게 연결되는 방법을 제공한다. 
## Usage
`SimpleRouter`를 이용한 간단한 URL conf 예제.

```python
from rest_framework import routers

router = routers.SimpleRouter()
router.register(r'users', UserViewSet)
router.register(r'accounts', AccountViewSEt)
urlpatterns = router.urls
```

`register()` 메서드에는 두 가지의 필수 argument가 있다. 

- `prefix`  
	경로에 사용할 URL prefix
- `viewset`  
	viewset 클래스
	
선택적으로 추가 argument를 지정할 수 있다. 

- `base_name`  
	작성된 URL 이름에 사용된다. 설정하지 않으면 basename은 viewset의 `queryset` 속성을 기반으로 자동 생성된다. viewset에 `queryset` 속성이 포함되어있지 않으면 viewset을 등록할 때 `base_name`을 설정해야한다. 
	
위의 예제는 다음과 같은 URL 패턴을 생성한다. 


URL pattern: `^users/$`  Name: `'user-list'`  
URL pattern: `^users/{pk}/$`  Name: `'user-detail'`  
URL pattern: `^accounts/$`  Name: `'accounts-list'`  
URL pattern: `^accounts/{pk}$`  Name: `'accounts-detail'`  


**Note:**  
`base_name` argument는 view name  패턴의 초기 부분을 지정하는데 사용된다. 위의 예제에서는 `user`와 `account` 부분에 해당한다. 

일반적으로 `base_name` argument를 지정할 필요는 없지만 커스텀 `get_queryset` 메서드를 정의한 viewset의 경우, `.queryset` 속성 set이 없을 수 있다. 해당 viewset을 등록하려고 하면 다음과 같은 오류를 보게된다. 

```python
'base_name' argument not sepcified, and could not automatically determine the name from the viewset, as it does not have a '.queryset' attribute.
```
모델 이름에서 자동으로 결정할 수 없으므로 viewset을 등록할 때 `base_name` argumnet를 명시적으로 설정해야한다. 

### Using `include` with routers
router 인스턴스의 `.urls` 속성은 URL 패턴의 표준 리스티일 뿐이다. 이런 URL을 포함하는 방법에는 여러가지 스타일이 있다. 

예를들어 `router.urls`를 기존의 view 리스트에 추가할 수 있다. 

```python
router = routers.SimpleRouter()
router.register(r'users', UserViewSet)
router.register(r'accounts', AccountViewSet)

urlpatterns = [
	url(r'^forgot-password/$', ForgotPasswordFormView.as_view())
]

urlpatterns += router.urls
```

Django의 include 함수를 사용할 수도 있다. 

```python
urlpatterns = [
	url(r'^forgot-password/$', ForgotPasswordFormView.as_view()),
	url(r'^', include(router.urls)),
]
```

router URL 패턴도 namespace가 될 수 있다. 

```python
urlpatterns = [
	url(r'^forgot-password/$', ForgotPasswordFormView.as_view()),
	url(r'^api/', include(router.urls, namesapce='api')),
]
```

하이퍼링크가 있는 serializer와 함께 namespace를 사용하는 경우 serializer의 `view_name` argument가 namespace를 올바르게 반영하는지 확인해야한다. 위의 예제에서 user detail view에 하이퍼링크된 serializer 필드에 대해 `view_name='api:user-detail'`과 같은 매개 변수를 포함해야한다. 

### Extra link and actions
`@detail_route`나 `@list_route`로 데코레이트된 viewset의 모든 메서드도 라우팅된다. 예를들어  `UserViewSet`클래스에서 다음과 같이 메서드들이 주어진 경우,

```python
from myapp.permissions import IsAdminOrIsSelf
from rest_framework.decorators import detail_route

class UserViewSet(ModelViewSet):
	...
	@detail_route(methods=['post'], permission_classes=[IsAdminOrIsSelf])
	def set_password(self, request, pk=None):
	...
```

다음과 같은 URL 패턴이 추가로 생성된다. 

- URL pattern: `^users/{pk}/set_password/$` Name: `user-set-password`

커스텀 동작에 대해 생성된 기본 URL을 사용하지 않으려면, ulr_path 매개변수를 사용해 커스텀할 수 있다. 

예를들어 커스텀 동작의 URL을 `^users/{pk}/change-password/$`로 변경하려면 다음과 같이 작성할 수 있다. 

```python
from myapp.permissions import IsAdminOrIsSelf
from rest_framework.decorators import detail_route

class UserViewSet(ModleViewSet):
	...
	@detail_route(methdos=['post'],
		permission_classes=[IsAdminOrIsSelf],
		url_path='change-password')
	def set_password(self, request, pk=None):
```
위의 예제는 다음과 같은 URL패턴을 생성한다. 

- URL pattenr : `^users/{pk}/change-password/$` Name: `'user-change-password'`

또한 url_path 및 url_name 매개변수를 함께 사용해 커스텀 동작에 대한 URL 생성을 제어할 수 있다. 

더 자세한 내용은 [marking extra actions for routing](http://www.django-rest-framework.org/api-guide/viewsets/#marking-extra-actions-for-routing) 참고

# API Guide
이 router 에는 `list`, `create`, `retrieve`, `update`, `partial_update`, `destroy` 작업에 대한 경로가 포함된다. viewset은 `@detail_route` 또는 `@list_route` 데코레이터를 사용해 라우트 될 추가 메서드를 표시 할 수 있다. 

<table>
	<tr>
		<td>URL Style</td>
		<td>HTTP Method</td>
		<td>Action</td>
		<td>URL Name</td>
	</tr>
	<tr>
		<td rowspan="2">{prefix}/</td>
		<td>GET</td>
		<td>list</td>
		<td rowspan="2">{basename}--list</td>
	</tr>
		<td>POST</td>
		<td>create</td>
	<tr>
	</tr>
	<tr>
		<td>{prefix}/{methodname}/</td>
		<td>GET, or as specified by `methods` argument</td>
		<td>`@list_route` decorated method	</td>
		<td>{basename}-{methodname}
</td>
	</tr>
	<tr>
		<td rowspan="4">{prefix}/{lookup}/</td>
		<td>GET</td>
		<td>retrieve</td>
		<td rowspan="4">{basename}-detail</td>
	</tr>
	<tr>
		<td>PUT</td>
		<td>update</td>
	</tr>
	<tr>
		<td>PATCH</td>
		<td>aprtail_update</td>
	</tr>
	<tr>
		<td>DELTE</td>
		<td>destroy</td>
	</tr>
	<tr>
		<td>{prefix}/{lookup}/{methodname}/	</td>
		<td>GET, or as specified by `methods` argument</td>
		<td>`@detail_route` decorated method</td>
		<td>{basename}-{methodname}
</td>
	</tr>
</table>

기본적으로 `SimpleRouter`로 만든 URL 뒤에는 슬래시가 추가된다. 이 동작은 router를 인스턴스화 할 때 `trailing_slash` argument를 `False`로 설정해 수정할 수 있다. 

```pythno
router = SimpleRouter(trailing_slash=False)
```

trailing slash 는 Django에서 일반적이지만 Rails와 같은 다른 프레임워크에서는 기본적으로 사용하지 않는다. 

router는 슬래시와 마침표를 제외한 문자가 포함된 조회 값을 일치시킨다. 더 제한적이거나 관대한 검색 패턴의 경우, viewset에 `lookup_value_regex` 속성을 설정해 사용한다. 예를들어 lookup을 유효한 UUID로 제한할 수 있다. 

```python
class MyModelViewSet(mixins.RetrieveModelMixin, viewsets.GenericViewSet):
	lookup_field = 'my_modle_id'
	lookup_value_regex = '[0-9a-f]{32}'
```

## SimpleRouter
## DefaultRouter

# Custom Routers
## Customizing dynamic routes
## Advanced custom routers