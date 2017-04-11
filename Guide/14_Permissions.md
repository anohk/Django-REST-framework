# Permissions
인증 및 제한과 함께 사용권한은 요청에 접근을 허용할지 거부할지 결정한다. 

권한 검사는 다른 코드가 진행되기 전, 항상 view의 맨 처음에 실행된다. 권한 검사는 일반적으로 들어오는 요청을 허용해야하는지 결정하기 위해 `request.user`및  `request.auth` 등록 정보의 인증 정보를 사용한다. 

권한은 다른 클래스의 사용자가 API의 다른 부분에 접근하는 것을 허용하거나 거부하는데 사용된다. 

가장 간단한 사용권한 스타일은 인증된 사용자에게 접근을 허용하고, 인증되지 않은 모든 사용자에 대해 접근을 거부하는 것이다. 이는 REST framework의 `IsAuthenticated` 클래스에 해당한다. 

조금 덜 엄격한 권한 스타일은 인증된 사용자에게 모든 권한을 허용하지만, 인증되지 않은 사용자에게는 읽기 전용 권한을 허용하는 것이다. 이것은 REST framework의 `IsAuthenticatedOrReadOnly` 클래스에 해당한다.

## How permissions are determined
REST framework의 권한은 항상 권한 클래스 목록으로 정의된다. 

view의 본문을 실행하기 전 목록의 각 권한을 검사한다. 권한 확인에 실패하면 `exceptions.PermissionsDenied` 또는 `exceptions.NotAuthenticated` 예외가 발생하고 view가 실행되지 않는다. 

권한 검사에 실패하면 다음 규칙에 따라 '403 Fobidden' 또는 '401 Unauthorized' 응답이 반환된다. 

**`HTTP 403 Forbidden`**  

- 요청이 성공적으로 인증되었지만 권한이 거부된 경우
- 요청이 성공적으로 인증되지 않았고, 최상위 우선순위 인증 클래스는 `WWW-Authenticate` 헤더를 사용하지 않은 경우

**`HTTP 401 Unauthorized`**  

- 요청이 성공적으로 인증되지 않았고 최상위 우선순위 인증 클래스가 `WWW-Authenticate` 헤더를 사용하는 경우 적절한 `WWW-Authenticate` 헤더가 있는 HTTP 401 Unauthorized 응답을 반환한다.

## Object level permissions
REST framework 권한은 object-level 권한 부여를 지원한다. object-level 권한은 사용자가 특정 object(일반적으로 모델 인스턴스)에 대한 작업을 허용해야하는지 여부를 결정하는데 사용된다. 

object-level 권한은 `.get_object()` 가 호출될 때 REST framework의 generic view에 의해 실행된다. view level의 접근 권한의 경우처럼, 사용자가 지정된 object를 조작할 수 없는 경우 `exceoptions.PermissionsDenied` 예외가 발생한다. 

obeject-level 권한에 대해 커스텀하려면 `get_object` 메서드를 오버라이드하고, `.check_object_permissions(request, obj)` 메서드를 명시적으로 작성해야한다. 

view에 적절한 권한이 있는 경우에는 반환되지만 그렇지 않으면 `PermissionDenied` 또는 `NotAuthenticated` 예외가 발생한다.

```python
def get_object(self):
	obj = get_object_or_404(self.get_queryset())
	self.check_object_permissions(self.request, obj)
	return obj
```

## Setting the permission policy

기본 권한 정책은 `DEFAULT_PERMISSION_CLASSES` 설정을 사용해 전역으로 설정할 수 있다. 

```python
REST_FRAMEWORK = {
	'DEFAULT_PERMISSION_CLASSES': (
		'rest_framework.permissions.IsAuthenticated',
	)
}
```
만약 지정하지 않으면 설정은 기본적으로 무제한 접근을 허용한다. 

```python
'DEFUALT_PERMISSION_CLASSES': (
	'rest_framework.permissions.AllowAny',
)
```

또한 `APIView` 클래스 기반 view를 사용해 view 단위 인증 정책을 설정할 수 있다. 

```python
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
	permission_classes = (IsAuthenticated,)
	
	def get(self, request, format=None):
		content = {
			'status': 'request was permitted'
		}
		return Response(content)
```

또는 `@api_view` 데코레이터가 적용된 함수 기반 view에서 사용할 수 있다. 

```python
from rest_framework.decorators import api_view, permissions_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response

@api_view(['GET'])
@permission_classes((IsAuthenticated,))
def example_view(request, format=None):
	content = {
		'status': 'request was permitted'
	}
	return Response(content)
```

**NOTE**  
클래스 속성이나 데코레이터를 통해 새 권한 클래스를 설정하면 settings.py 파일을 통해 설정된 기본 목록을 무시하도록 view에 지시한다. 

# API Reference
## AllowAny
`AllowAny` 권한 클래스는 요청이 인증되었는지 아닌지 여부와 관계없이 제한되지 않은 접근을 허용한다. 

사용권한 설정에 빈 목록이나 튜플을 사용해 동일한 결과를 얻을 수 있기 때문에 사용권한은 반드시 필요한 것은 아니지만 의도를 명시적으로 보여주기 때문에 지정하는 것이 유용하다. 

## IsAuthenticated
`IsAuthenticated` 권한 클래스는 인증되지 않은 사용자에게 권한을 거부하고 그렇지 않은 경우 허용한다. 

이 권한은 등록된 사용자만 API에 접근할 수 있게 하려는 경우 적합하다. 

## IsAdminUser
`IsAdminUser` 권한 클래스는  `user.is_staff`가 `True`인 경우를 제외하고 모든 사용자에 대한 사용권한을 거부한다. 

이 권한은 신뢰할 수 있는 관리자의 하위 집합에서만 API를 접근할 수 있게 하려는 경우 적합하다. 

## ISAuthenticatedOrReadOnly
`IsAuthenticatedOrReadOnly`는 인증된 사용자가 모든 요청을 수행할 수 있다. 권한이 없는 사용자에 대한 요청은 요청 메서드가 안전한 경우에만 허용한다. (GET, HEAD, OPTIONS와 같은 경우)

이 권한은 API에서 익명 사용자에게 읽기 권한을 허용하고 인증된 사용자에게만 쓰기 권한을 허용하려는 경우에 적합하다. 

## DjangoModelPermissions
이 권한 클래스는 Django의 표준 django.contrib.auth 모델 권한과 관련이 있다. 이 권한은 `.queryset` 속성이 설정된 view 에만 적용해야한다. 권한 부여는 사용자가 인증되고 관련모델 권한이 할당된 경우에만 부여된다. 

- `POST` 요청을 하려면 사용자에게 해당 모델에 관한 `add` 권한이 있어야한다. 
- `PUT`, `PATCH` 요청을 하려면 사용자에게 해당 모델에 관한 `change` 권한이 있어야한다. 
- `DELETE` 요청을 하려면 사용자에게 해당 모델에 관한 `delete` 권한이 있어야한다. 

기본 동작을 커스텀하여 커스텀 모델 권한을 지원할 수도 있다. 예를 들어 `GET` 요청에 대한 `view` 모델 권한을 포함할 수 있다. 

커스텀 모델 권한을 사용하려면 `DjangoModelPermissions`를 오버라이드하고 `.perms_map` 속성을 설정해야한다. 

### Using with views that do not include a `queryset` attribute
오버라이드된 `get_queryset()` 메서드를 사용하는 view에 이 권한을 사용하는 경우 view에 `queryset`속성이 없을 수 있다. 이 경우 sentinel queryset으로 view를 표시해 이 클래스가 필요한 권한을 결정할 수 있도록 하는 것이 좋다. 

```python 
queryset = User.objects.non()
```

## DjangoModelPermissionsOrAnonReadOnly
DjangoModelPermissions와 유사하지만 인증되지 않은 사용자는 API에 대한 읽기 전용 접근만 허용한다. 

## DjangoObjectPermissions
이 권한 클래스는 모델에 대한 객체 별 권한을 허용하는 Django의 표준 object permissions framework와 관련이 있다. 이 권한 클래스를 사용하려면 django-guardian과 같은 object-level을 지원하는 권한 백엔드를 추가해야한다. 

`DjangoModelPermissions`와 마찬가지로 이 권한은 `.queryset` 속성이나 `.get_queryset()` 메서드가 있는 view에만 적용되어야한다. 사용자가 인증되과 관련 객체 별 권한 및 관련 모델 권한이 할당 된 경우에만 부여된다. 

- `POST` 요청을 하려면 사용자에게 해당 모델 인스턴스에 관한 `add` 권한이 있어야한다. 
- `PUT`, `PATCH` 요청을 하려면 사용자에게 해당 모델 인스턴스에 관한 `change` 권한이 있어야한다. 
- `DELETE` 요청을 하려면 사용자에게 해당 모델 인스턴스에 관한 `delete` 권한이 있어야한다.

`DjangoObjectPermissions`는 `django-guardian` 패키지를 필요로하지 않으며 다른 object-level 백엔드도 똑같이 잘 지원해야한다. 

`DjangoModelPermissions`와 마찬가지로 `DjangoObjectPermissions`를 오버라이드하고 `.perms_map` 속성을 설정하여 커스텀 모델 권한을 사용할 수 있다. 


# Custom Permissions
커스텀 접근 권한을 구현하려면, `BasePermissions` 를 오버라이드하여 다음과 같은 메서드 중 하나 이상 구현한다.

- `.has_permission(self, request, view)`
- `.has_objects_permission(sefl, request, view, obj)`

요청에 접근 권한이 부여되면 메서드는 `True`를 반환하고 그렇지 않으면 `False`를 반환해야한다. 

요청이 읽기 작업인지 아니면 쓰기 작업인지 테스트해야하는 경우 `GET`, `OPTIONS` 및 `HEAD`가 포함된 튜플인 `SAFE_METHODS` 상수와 비교하여 요청 메서드를 확인해야한다. 

```python
if request.method in permissions.SAFE_METHODS:
	# Check permissions for read-only reuqest
else: 
	# Check permissions for write request
```

**NOTE**  
view-levle `has_permission` 검사가 이미 통과된 경우에만 instance-level의 `has_object_permission` 메서드가 호출된다. 
또한 instance-level 검사를 실행하려면 view 코드에서 `.check_object_permissions(request, obj`를 명시적으로 호출해야한다. generic view를 사용하는 경우 기본적으로 이 옵션은 처리된다. 


테스트가 실패할 경우 커스텀 권한은 `PermissionDenied` 예외를 발생시킨다. 예외와 관련된 오류 메세지를 변경하려면 커스텀 권한에 직접 메세지 속성을 구현해야한다. 그렇지 않으면 `PermissionDenied`의 `default_detail` 속성이 사용된다. 

```python
from rest_framework import permissions

class CustomerAccessPermission(permissions.BasePermission):
	message = 'Adding customers not allowed.'
	
	def has_permissions(self, request, view):
		...
```

## Examples
다음은 들어오는 요청의 IP주소를 블랙리스트와 대조하여 IP가 블랙리스트에 올라와있으면 요청을 거부하는 권한 클래스의 예이다. 

```python
from rest_framework import permissions

class BlacklistPermission(permissions.BasePermission):

	def has_permission(self, request, view):
		ip_addr = request.META['REMOTED_ADDR']
		blacklisted = Blacklist.objects.filter(ip_addr=ipaddr).exists()
		return not blacklisted
```

들어오는 모든 요청에 대해 실행되는 전역 권한 뿐 아니라 특정 인스턴스에 영향을 주는 작업에 대해서만 실행되는 object-level 권한을 만들 수도 있다. 

```python
class IsOwnerOrReadOnly(permissions.BasePermission):
	
	def has_object_permission(self, request, view, obj):
	
	# Read permissions are allowed to any request,
    # so we'll always allow GET, HEAD or OPTIONS requests.
	if request.method in permissions.SAFE_METHODS:
		return True
		
    # Instance must have an attribute named `owner`.
	return obj.owner == request.user
```

generic view는 적절한 object-level 권한을 검사하지만 커스텀의 경우 권한 검사를 직접 확인해야한다. 객체 인스턴스가 있으면 view에서 `self.check_object_permissions(request, obj)`를 호출하면 된다. object-level의 접근 권한 검사가 실패했을 경우, 이 호출에 의해 적절한 `APIException` 이 발생한다. 

또한 generic view는 단일 모델 인스턴스를 검색하는 view에 대한 object-level 권한만 검사한다. list view의 object-level 필터링이 필요한 경우 별도로 쿼리셋을 필터링해야한다. 
