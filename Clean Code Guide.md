
# 1. Clean Code Guidelines
https://github.com/Yooii-Studios/Clean-Code

## 클래스 이름

명사 혹은 명사구를 사용하라.(Customer, WikiPage, Account, AddressParser).
Manager, Processor, Data, Info와 같은 단어는 피하자.
동사는 사용하지 않는다.


## 메서드 이름
동사 혹은 동사구를 사용하라.(postPayment, deletePayment, deletePage, save 등)
접근자, 변경자, 조건자는 get, set, is로 시작하자. (추가: should, has 등도 가능)
생성자를 오버로드할 경우 정적 팩토리 메서드를 사용하고 해당 생성자를 private으로 선언한다.

## 한 개념에 한 단어를 사용하라
추상적인 개념 하나에 단어 하나를 사용하자.
fetch, retrieve, get
controller, manager, driver

##의미 있는 맥락을 추가하라
클래스, 함수, namespace등으로 감싸서 맥락(Context)을 표현하라
그래도 불분명하다면 접두어를 사용하자.

## 함수를 만들 때 최대한 ‘작게!’ 만들어라.
한 함수당 3~5줄 이내로 줄이는 것을 권장한다

함수 내 섹션
함수를 여러 섹션으로 나눌 수 있다면 그 함수는 여러작업을 하는 셈이다.

## “코드를 읽으면서 짐작했던 기능을 각 루틴이 그대로 수행한다면 깨끗한 코드라 불러도 되겠다” - 워드

##단항 함수는 함수와 인수가 동사/명사 쌍을 이뤄야한다.
writeField(name);
함수이름에 키워드(인수 이름)을 추가하면 인수 순서를 기억할 필요가 없어진다.
assertExpectedEqualsActual(expected, actual);



## TODO 주석은 프로그래머가 필요하다 여기지만 당장 구현하기 어려운 업무를 기술한다. 더 이상 필요 없는 기능을 삭제하라는 알림, 누군가에게 문제를 봐달라는 요청, 더 좋은 이름을 떠올려달라는 부탁, 앞으로 발생할 이벤트에 맞춰 코드를 고치라는 주의 등에 유용하다. 요즘은 IDE를 통해 남은 TODO를 쉽게 볼 수 있으므로 편리하게 이용할 수 있다.











