# Serializers

Serializers는 querysets 및 모델 인스턴스와 같은 복잡한 데이터를 `JSON`, `XML` 또는 기타 콘텐츠 타입으로 쉽게 렌더링할 수 있는 native Python 데이터 타입으로 변환할 수 있도록 한다. 또한 serializer는 deserialization을 제공해 들어오는 데이터의 유효성을 처음 확인한 후에 구문분석된 데이터를 복합 형식으로 다시 변환할 수 있다. 

REST framework는 Django의 Form및 ModelForm 클래스와 매우 유사하게 작동한다. 
응답의 출력을 제어하는 강력하고 일반적인 방법을 제공하는 `Serializer` 클래스를 제공한다. 또한 `ModelSerializer` 클래스 뿐만 아니라 모델 인스턴스 및 queryset을 처리하는 serializer를 만드는데 유용한 shortcut을 제공한다. 

## Declaring Serializers

예제를 위해 사용할 간단한 객체를 만들어본다. 

```python
from datetime import datetime

class Comment(object):
	def __init__(self, email, content, crated=None):
		self.email = email
		self.content = content
		self.created = created or datetime.now()
		
comment = Comment(email='leila@example.com', content='foo bar')
```

`Commnet` 객체에 대한 데이터를 serialize하고 deserialize할 수 있는 serializer를 선언한다. 

```python
from rest_framework import serializers

class CommentSerializer(serializers.Serializer):
	email = serializers.EmailField()
	content = serializers.CharField(max_length=200)
	created = serializers.DateTimeField()
```

## Serializing objects
comment 또는 comment의 리스트를 직렬화하기 위해 `CommentSerializer`를 사용할 수 있다. 

```python
serializer = CommentSerializer(comment)
serializer.data
# {'email': 'leila@example.com', 'content': 'foo bar', 'created': '2016-01-27T15:17:10.375877'}
```
모델 인스턴스를 Python native 데이터 타입으로 변환했다. 직렬화를 마무리하기 위해 데이터를 `json`으로 렌더링한다. 

```python
from rest_framework.renderers import JSONRenderer

json = JSONRenderer().render(serializer.data)
json
# b'{"email":"leila@example.com","content":"foo bar","created":"2016-01-27T15:17:10.375877"}'
```

## Deserializing objects
비직렬화도 비슷한다. 먼저 Python native 데이터타입으로 stream을 분석한다. 

```python
from django.utils.six import BytesIO
from rest_framework.parsers import JSONParser

stream = BytesIO(json)
data = JSONParser().parse(stream)
```
그리고 유효성이 검사된 dictionary로 저장한다. 

```python
serializer = CommentSerializer(data=data)
serializer.is_valid()
# True
serializer.validated_data
# {'content': 'foo bar', 'email': 'leila@example.com', 'created': datetime.datetime(2012, 08, 22, 16, 20, 09, 822243)}
```

## Saving instances
유효성이 검사된 데이터를 기반으로 완전한 객체 인스턴스를 반환하려면 `.create()`와 `.update()`메서드 중 하나 또는 둘 모두를 구현해야한다. 

```python
class CommentSerializer(serializers.Serializer):
	email = serializers.EmailField()
	content = serialiizers.CharField(max_length=200)
	created = serializers.DateTimeField()
	
	def create(self, validated_data):
		return Comment(**validated_data)
	
	def update(self, instance, validated_data):
	instance.email = validated_data.get('email', instance.email)
	instance.content = validated_data.get('content', instance.content)
	instance.created = validated_data.get('created', instance.created)
	return instance
```

객체 인스턴스가 Django 모델과 일치하는 경우, 메서드가 객체를 데이터베이스에 저장하도록 해야한다. 예를들어, `Comment`가 Django 모델인 경우 메서드는 다음과 같이 작성할 수 있다. 

```python
def create(self, validated_data):
	return Comment.objects.create(**validated_data)

def update(self, instance, validated_data):
	instance.email = validated_data.get('email', instance.email)
	instance.content = validated_data.get('content', instance.content)
	instance.created = validated_data.get('created', instance.created)
	instance.save()
	return instance
```

데이터를 비 직렬화 할 때 `.save()`를 호출해 유효성이 검사된 데이터를 기반으로 객체 인스턴스를 반환할 수 있다. 

```python
comment - serializer.save()
```

`.save()` 메서드를 호출하면 serializer 클래스를 인스턴스화 할 때, 인스턴스의 존재 여부에 따라 인스턴스를 생성하거나 업데이트한다. 

```python
# .save() will create a new instance.
serializer = CommentSerializer(data=data)

# .save() will update the existing 'comment' instance.
serializer = CommentSerializer(comment, data=data)
```

### Passing additional attributes to `.save()`
`.save()` 메서드를 호출할 때 추가적인 키워드 argument를 사용할 수 있다. 

```python
serializer.save(owner=request.user)
```

추가 키워드 argument는 `.create()`나 `.update()` 가 호출될 때, `validated_data` argument에 추가된다. 

### Overriding `.save()` directly
어떤 경우에는 `.create()`나 `.update()` 메서드 이름이 의미가 없을 수 있다. 

예시: 연락처 양식에 새로운 인스턴스를 만들지 않고 전자메일 또는 메세지를 보낸다. 이 경우 `.save()`메서드를 직접 읽고 무시할 수 있다. 

```python
class ContactForm(serializers.Serializer):
	email = serializers.EmailField()
	message = serializers.CharField()
	
	def save(self):
		email = self.validated_data['email']
		message = self.validated_data['message']
		send_email(from=email, message=message)
```
위와 같은 경우 `.validated_data`에 직접 접근해야한다. 

## Validation
비직렬화를 할 때, validated data에 접근하기 전에 `is_valide()`를 호출하거나 객체 인스턴스를 저장해야한다. 유효성 검사에서 오류가 발생하면 `.errors` 속성에는 오류메세지를 나타내는 dictionary가 포함된다.  

```python
serializer = commentSerializer(data={
				'email': 'foobar'
				'content': 'baz'
				})
serializer.is_valid()
# False
serializer.errors
# {'email': [u'Enter a valid e-mail address.'], 'created': [u'This field is required.']}
```

### Raising an exception on invalid data
`.is_valid()` 메서드는 유효성 검사 오류가 있는 경우 `serializers.ValidationError` 예외를  발생시키는 선택적 `raise_exception` 플래그를 사용한다. 

이런 예외는 REST framework에서 제공하는 기본 예이 처리기에서 자동으로 처리되며 기본적으로 `HTTP 400 Bad Request`를 리턴한다. 

```python
# Return a 400 response if the data was invalid.
serializer.is_valid(raise_exception=True)
```

### Field-level validation
### Object-level validation
### vAlidators

## Accessing the initial data and instance
serializer 인스턴스에 초기 객체 또는 queryset을 전달할 때, 객체는 `.instance`를 사용한다. 초기 객체가 전달되지 않으면 `.instance` 속성은 `None`이된다. 

serializer 인스턴스를 전달할 때, 수정되지 않은 데이터는 `.initial_data`로 사용할 수 있다. data 키워드 argument가 전달되지 않으면 `.initial_data`는 존재하지 않는다.

## Partial updates
기본적으로 serializer는 모든 필수 필드에 값을 전달해야하며 그렇지 않으면 유효성 검사 오류가 발생한다. `partial` argument를 사용한다면 부분적으로 업데이트할 수 있다. 

```python
# Update 'comment' with partial data
serializer = CommentSerializer(comment, data={'content': u'foo bar'}, partial=True)
```

## Dealing with nested objects

`serializer`클래스는 자체는 `Field` 타입이고, 한 객체 유형이 다른 객체 내에 중첩되어있는 관계를 나타내는데 사용할 수 수 있다. 

```python
class UserSerializer(serializers.Serializer):
	email = serializers.EmailField()
	username = serializers.CharField(max_length=100)
	
class CommentSerializer(serializers.Serailizer):
	user = UserSerializer()
	content = serializers.CharField(max_length=200)
	created = serializers.DateTiemField()
```

중첩된 표현이 `None`값을 선택적으로 받아들일 수 있으면 `required=False` 플래그를 중첩된 serializer에 전달해야한다. 

```python
class CommentSerializer(serializers.Serailizer):
	user = UserSerializer(required=False)
	content = serializers.CharField(max_length=200)
	created = serializers.DateTimeField()
```

중첩된 표현이 아이템의 리스트이어야한다면, 중첩된 serializer에 `many=True` 플래그를 전달해야한다. 

```python
class CommentSerializer(serializers.Serializer):
	user = UserSerializer(required=False)
	edits = EditItemSerializer(many=True) # A nested list of 'edit' items.
	content = serializers.CharField(max_length=200)
	created = serializers.DateTimeField()

```

## Writable nested representations
데이터의 비 직렬화를 지원하는 중첩된 표현을 처리할 때 중첩된 객체의 오류는 중첩된 객체의 필드이름 아래에 중첩된다. 

```python
serializer = CommentSerializer(data={'user':{'email: 'foobar', 'usernmae': 'doe'}, 'content': 'baz'})
serializer.is_valid()
# False
serializer.errors
# {'user': {'email': [u'Enter a valid e-mail address.']}, 'created': [u'This field is required.']}
```

이와 유사하게 `.validated_data` 속성은 중첩된 데이터 구조를 포함한다. 

### Writing `.create()` methods for nested representations

쓰기 가능한 중첩표현을 지원하려면 여러 객체를 저장하는 `.create()` 또는 `.update()` 메서드를 작성해야한다. 

아래의 예제는 중첩된 프로필 객체가 있는 사용자 생성을 처리하는 방법을 보여준다. 

```python 
class Userserializer(serializers.ModelSerializer):
	profile = ProfileSerializer()
	
	class Meta:
		modle = User
		fields = ('username', 'email', 'profile')
		
	def create(self, validated_data):
		profile_data = validated_data.pop('profile')
		user = User.objects.create(**validated_data)
		Profile.objects.create(user=uesr, **profile_data)
		return user
```

### Writing `.update()` methdos for nested representations

업데이트를 하는 경우, 관계 업데이트 처리에 대해 신중하고 싶을 것이다. 예를들어 관계에 대한 데이터가 `None` 또는 제공되지 않은 경우 다음 중 어떤 것이 발생해야할까?

- 데이터베이스에서 관계를 `Null`로 설정한다.
- 관련된 인스턴스를 삭제한다.
- 데이터를 무시하고 인스턴스를 그대로 둔다.
- validation error를 발생시킨다. 

다음은 이전 `UserSerializer` 클래스의 `update()` 메서드 예제이다. 

```python
def update(self, instance, validated_data):
	profile_data = validated_data.pop('profile')
	profile = instance.profile
	
	instance.username = validated_data.get('username', instance.username)
	instance.email = validated_data.get('email', instance.email)
	instance.save()
	
	profile.is_premium_member = profile_data.get('is_premium_member', profile.is_premium_member)
	profile.has_support_contract = profile_data.get('has_support_contract', profile.has_support_contract)
	profile.save()
	
	return instance
```
중첩된 생성 및 업데이트의 동작이 모호할 수 있고 관련 모델 간의 복잡한 종속성이 필요할 수 있기때문에 REST framework3 에서는 이러한 메서드를 항상 명시적으로 작성해야한다. 기본 `ModelSerializer`의 `.create()`, `.update()` 메서드는 쓰기 가능한 중첩표현에 대한 지원을 하지 않는다. 

### Handling saving related instances in model manager classes

serializer에 여러 관련 인스턴스를 저장하는 대신 올바른 인스턴스를 생성하는 커스텀 모델 관리자 클래스를 작성할 수 있다. 

예를들어 `User` 인스턴스와 `Profile` 인스턴스가 항상 쌍으로 함께 생성되도록 하고 싶다면 다음과 같이 커스텀 매니저 클래스를 작성할 수 있다. 

```python
class UserManager(models.Manager):
	...
	def create(self, username, email, is_premium_member=False, has_support_contract=False):
		user = User(username=username, email=email)
		user.save()
		profile = Profile(
			user=user,
			is_premium_member=is_premium_member,
			has_support_contract=has_support_contract
		)
		profile.save()
		return user
```

이 관리자 클래스는 이제 user 인스턴스와 profile 인스턴스가 항상 동시에 생성된다는 사실을 캡슐화한다. 
serializer 클래스의 `.create()` 메서드는 새 관리자 메서드를 사용할 수 있도록 다시 작성할 수 있다. 

```python
def create(self, validated_data):
	return User.objects.create(
		username=validated_data['username'],
		email=validated_data['email'],
		is_premium_member =validated_data['profile']['is_premium_member']
		has_support_contract=validated_data['profile']['has_support_contract']
	)
```

더 자세한 것은 Django 문서인 [model managers](https://docs.djangoproject.com/en/1.10/topics/db/managers/) 참고.

## Dealing with multiple objects

`Serializer` 클래스는 객체 목록의 직렬화 또는 비 직렬화를 처리 할 수도 있다. 

### Serializing multiple objects
단일 객체 인스턴스 대신 객체의 목록이나 queryset을 직렬화하려면, serializer를 인스턴스화 할 때 `many=True` 플래그를 넘겨줘야한다. 그리고 나서 직렬화 할 queryset또는 객체 목록을 전달 할 수 있다. 

```python
queryset = Book.objects.all()
serializer = BookSerializer(queryset, many=True)
serializer.data
# [
#     {'id': 0, 'title': 'The electric kool-aid acid test', 'author': 'Tom Wolfe'},
#     {'id': 1, 'title': 'If this is a man', 'author': 'Primo Levi'},
#     {'id': 2, 'title': 'The wind-up bird chronicle', 'author': 'Haruki Murakami'}
# ]
```

### Deserializing multiple objects
여러 객체를 비 직렬화 하는 기본 동작은 여러 객체 생성을 지원하지만 여러 객체 업데이트는 지원하지 않는다. 커스텀 방법은 [ListSerializer](http://www.django-rest-framework.org/api-guide/serializers/#listserializer)참고 

## Including extra context
직렬화되는 객체 외에도 추가 컨텍스트를 직렬화에 제공해야할 경우가 있다. 한 가지 일반적인 경우는 하이퍼링크 된 관계를 포함하는 serializer를 사용하는 것이며, serializer가 현재 요청에 접근해 정규화된 URL을 제대로 생성할 수 있어야한다. 

serializer를 인스턴스화 할 때 `context` argument를 전달해 임의의 추가 컨텍스트를 제공할 수 있다. 

```python
serializer = AccountSerializer(accoutn, context={'request': request})
serializer.data
# {'id': 6, 'owner': u'denvercoder9', 'created': datetime.datetime(2013, 2, 12, 09, 44, 56, 678870), 'details': 'http://example.com/accounts/6/details'}
```

context dictionary는 custom `.to_representation()` 메서드와 같은 serializer 필드 로직 내에서 `self.context` 속성에 접근해 사용할 수 있다. 

# ModelSerializer
Django 모델 정의와 밀접하게 매핑되는 serializer 클래스가 필요할 때, `ModelSerializer`를 사용한다. 

`ModelSerializer` 클래스는 모델 필드에 해당하는 필드가 있는 `Serializer`클래스를 자동 생성하는 shortcut을 제공한다. 

`ModelSerializer`클래스는 아래의 내용을 제외하고는 일반적인 `Serializer`클래스와 유사하다. 

- 모델을 기반으로 일련의 필드가 자동으로 생성된다. 
- unique_together validator와 같은 serializer에 대한  validator를 자동으로 생성한다. 
- `.create()` 및  `.update()`의 간단한 기본 구현을 포함한다.

`ModelSerializer`를 정의하는 것은 다음과 같다. 

```python
class AccountSerializer(serializers.ModelSerializer):
	class Meta:
		model = Account
		fields = ('id', 'account_name', 'users', 'created')
```

기본적으로 클래스의 모든 모델 필드는 해당 serializer 필드에 매핑된다. 

모델의 foreignkey와 같은 관계는 `PrimaryKeyRelatedField`에 매핑된다. [serializer relations](http://www.django-rest-framework.org/api-guide/relations/) 문서에 명시된 것 처럼 역방향 관계는 기본적으로 포함되지 않기 때문에 명시적으로 포함시켜줘야 한다. 

### Inspecting a `ModelSerializer`
serializer 클래스는 유용한 verbose 표현 문자열을 생성한다. 이를 통해 해당 field들의 상태를 모두 조사할 수 있다. 이는 `ModelSerializer`로 작업할 때 자동으로 생성되는 필드 및 유효성 검사 set을 결정하는 부분에서  특히 유용하다. 

이렇게 하려면, `python manage.py shell`을 작성해 Django 쉘을 열고, serializer 클래스를 가져와 인스턴스한 후 객체 표현을 출력한다. 

```python
>>> from myapp.serializers import AccountSerializer
>>> serializer = Accountserializer()
>>> print(repr(serializer))
AccountSerializer():
    id = IntegerField(label='ID', read_only=True)
    name = CharField(allow_blank=True, max_length=100, required=False)
    owner = PrimaryKeyRelatedField(queryset=User.objects.all())
```

## Specifying which fields to include
모델 serializer에서 기본 필드의 하위 집합만 사용하려는 경우, `ModelForm`을 사용했던 것 처럼  `fields`나 `exclude`옵션을 사용할 수 있다. `fields` 속성을 사용해 직렬화해야하는 모든 필드를 명시적으로 설정하는 것이 좋다. 이렇게하면 모델이 변경 될 때 실수로 데이터가 노출 될 가능성이 줄어든다. 

```python
class AccountSerializer(serializers.ModelSerializer):
	class Meta:
		model = Account
		fields = ('id', 'account_name', 'users', 'created')
```

또한 `fields` 속성을 `__all__`로 설정해 모델의 모든 필드를 사용해야한다는 것을 나타낼 수도 있다. 

```python
class AccountSerializer(serializers.ModelSerializer):
	class Meta:
		model = Account
		fields = '__all__'
```

serializer에서 제외할 필드 목록을 `exclude` 속성으로 설정할 수 있다. 

```python
class AccountSerializer(serializers.ModelSerializer):
	class Meta:
		model = Account
		exclude = ('users',)
```
위의 예시에서 `Account` 모델에 `account_name`, `users`, `created` 필드 세 가지가 있는 경우, `account_name`과 `created` 필드가 생성되고 serialize 된다. 

`fields`와 `exclude` 속성의 이름은 일반적으로 모델 클래스의 모델 필드에 매핑된다. 

또는 `fields` 옵션의 이름은 모델 클래스에 존재하며 argument를 취하지 않는 속성이나 메서드에 매핑할 수 있다. 

## Specifying nested serialization
기본 `Modelseializer`는 관계에 primary key를 사용하지만 `depth` 옵션을 사용해 중첩된 표현을 쉽게 생성할 수도 있다. 

```python
class Accountserializer(serializers.modelSerializer):
	class Meta:
		model = Account
		fields = ('id', 'account_name', 'users', 'created')
		depth = 1
```
`depth` 옵션은 관계의 깊이를 나태내는 정수 값으로 설정해야한다. 

직렬화를 수행하는 방식을 커스텀하려면 필드를 직접 정의해야한다. 

## Specifying fields explicitly
`ModelSerializer`에 추가 필드를 추가하거나 `Serializer` 클래스에서와 마찬가지로 클래스의 필드를 선언해 기본 필드를 재정의할 수 있다. 

```python
class Accountserializer(serialziers.ModelSerializer):
	url = serializers.CharField(source='get_absolute_url', read_only=True)
	groups = serializers.PrimaryKeyRealtedField(many=True)
	
	class Meta:
		model = Account
```

추가 필드는 모델의 모든 속성 및 호출 가능한 항목에 해당할 수 있다. 

## Specifying read only fields
여러 필드를 읽기 전용으로 지정할 수 있다. 각 필드를 `read_only=True`로 명시적으로 추가하는 대신, shortcut으로 Meta option인 `read_only_fields`를 사용할 수 있다. 

이 옵션은 필드 이름의 list 또는 tuple이어야하며 다음과 같이 선언된다. 

```python
class AccountSerializer(serializers.ModelSerializer):
	class Meta:
		modle = Account
		fields = ('id', 'account_name', 'users', 'created')
		read_only_fields=('account_name',)
```
`editable=False` set이나 `AutoField` 필드가 있는 모델 필드는 기본적으로 읽기전용으로 설정되기때문에 `read_only_fields` 옵션에 추가 할 필요가 없다. 

**Note:**  
모델 레벨에서 읽기 전용 필드가 `unique_together` 제약 조건의 일부인 특별한 경우가 있다. 이 경우 필드는 제약 조건의 유효성을 검사하기 위해 serializer 클래스에서 필요하지만 사용자가 편집할 수 없어야한다. 

이러한 경우 `read_only=True` 및 `default = ...` 키워드 argument를 제공해 serializer에서 필드를 명시적으로 지정하면된다. 

현재 인증된 `User`에 대한 읽기전용 관계이며 다른 식별자와 `unique_together`인 경우 user 필드를 다음과 같이 정의한다. 

```python
user = serializers.PrimaryKeyRelatedField(read_only=True, default=serializers.CurrentUserDefault())
```

## Additional keyword arguments
## Relational fields
## Customizing field mappings
### The field_class and field_kwargs API

#HyperlinkedModelSerializer
`HyperlinkedModelSerializer` 클래스는 관계를 나타내기 위해 primary key가 아닌 하이퍼링크를 사용한다는 점을 제외하고는 `ModelSerializer`클래스와 유사하다. 

기본적으로 serializer에는 primary key 필드 대신 `url` 필드가 포함된다. 

url 필드는 `HyperlinkedIdentityField` serializer 필드를 사용하여 표현되고, 모델의 모든 관계는 `HyperlinkedRelatedField` serializer 필드를 사용해 표현된다. 

primary key를 `fields` 옵션에 추가해 명시적으로 포함시킬 수 있다. 

```python
class Accountserializer(serializers.HyperlinedModelSerializer):
	class Meta:
		model = Account
		fields = ('url', 'id', 'account_nmae', 'users', 'created')
```

## Absolute and relative URLs

## How hyperlined views are determined

## Changing the URL field name

# ListSerializer
# BaseSerializer
# Advanced serializer usage