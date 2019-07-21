# Redux-Saga Pattern

시작하기에 앞서 redux-saga에 대한 거의 모든 정보는 
https://redux-saga.js.org/ 공식문서에서 얻을 수 있다.

## 글 전체를 관통하는 원칙
- 일의 처리는 최대한 미룬다.
- 재활용 가능한 코드를 작성한다.
- 다형성 높은 함수를 작성한다. (인자가 적은 함수를 작성한다.)
- 함수의 합성에서 애러처리 룰은 KLEISLI ARROW에 근거한다.

## Redux-Saga의 역할
redux-saga(이하 사가)의 역할은 react/redux/redux-saga 의 스텍을 사용하는 어플리케이션에서 middle-ware를 담당한다. redux 3가지 원칙 중 하나인 `Changes are made with pure functions`에 근거하여 redux는 순수함수로 state의 변화를 만드는 역할을 한다. 그리고 사가는 redux가 맡고 있지 않은 side-effect 관리를 담당한다.

간단한 예를 들면 
```
button click -> api request -> api pending -> api resolve -> 상태 변경
```

이 예에서 사가는  api request 이후에서 부터 api resolve 까지의 역할을 맡는 것이 사가이다.

### 파일의 분리
redux/redux-saga는 큰 도메인 아레 saga, redux 각 파일을 생성한다. 예로 user/saga.js, user/type.js, user/action.js, user/reducer.js 를 작성한다. 필자는 redux ducks pattern(https://github.com/erikras/ducks-modular-redux)을 사용한다.

### 동기/비동기 이펙트(effect)로 만드는 패턴
사가에서 제공하는 기능들을 이펙트라 부르며 이 중에서 가장 많이 쓰이는 몇개의 이펙트를 설명하고 각 이펙트들의 특징에대해서 알아보겠다. 

#### take/call pattern

function* work() {
  ...
}

function* saga() {
  while(true) {
    const actions = yield take('TYPE');
    yield call(work, actions);
  }
}

사가의 가장 기본 패턴이라고 생각하는 코드이다. watch pattern이라고 볼 수 있는데 특정 액션이 발생하면 해당 액션 타입 을 'take' 하고 'call'로 handler 역할을 하는 task를 부른다. 'call'로 다른 제네레이터 함수를 부르면 불린 제네레이터, 즉 서브루틴은 그 루딘이 모두 끝나야 제네레이터의 next를 호출 가능하다. 

- 이 패턴을 사용해야 할 때와 사용하지 말아야 할 때
앞서 설명했듯이 'call'을 이용해 다른 제네레이터 함수를 부르기 때문에 work 함수가 일을 다 끝내기 전까지는 saga 함수는 액션 타입을 받을 수 없기 때문에 일련의 flow가 모두 끝나기 전에 동일한 flow를 시작하지 않기를 원할때 해당 패턴을 사용하면 된다. 반대로 여러번 요청이 와도 모두 처리해야 한다면 해당 패턴을 사용해선 안된다.

사용 가능 예. 사용자가 완료 버튼을 누를 때. 닫기 버튼을 여러번 누를 때.
사용 불가능 예. 다른 사가를 부르는 릴레이 역할을 하는 사가.




### redux-saga의 관심사
### 프로그램의 관심사
### 기획의 관심사
### 재사용 가능성
### 중복이란?
### 에러처리
#### try/catch
#### kslie error
### flow/work 패턴
### task 취소
### task 병렬처리
