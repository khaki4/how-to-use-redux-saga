# Redux-Saga Pattern

시작하기에 앞서 redux-saga에 대한 거의 모든 정보는 [공식문서](https://redux-saga.js.org/) 에서 얻을 수 있다.

## Prerequisite

redux-saga의 기본 [이펙트](https://redux-saga.js.org/docs/basics/DeclarativeEffects.html)를 사용 가능하며, sync/async, blocking/non-blocking 대하여 숙지한 자를 대상으로 한다.

## 글 전체를 관통하는 원칙

- 일의 처리는 최대한 뒤로 미룬다.
- 재활용 가능한 코드를 작성한다.
- 인자가 적은 함수가 인자가 많은 함수 보다 더 좋은 함수 이다.
- 함수의 합성에서 에러처리 룰은 KLEISLI COMPOSITION에 근거한다.

## Redux/Redux-Saga의 역할

redux는 3가지 원칙 중 하나인 ***Changes are made with pure functions*** 에 근거하여 redux는 순수함수로 state의 변화를 만드는 역할을 한다. redux-saga의 역할은 react/redux/redux-saga 의 스텍을 사용하는 어플리케이션에서 middle-ware를 담당, redux가 맡고 있지 않은 side-effect 관리한다.

간단한 예를 들면

```
button click --> api request --> api pending --> api resolve --> 상태 변경
```

이 예에서 사가는 api request 이후에서 부터 api resolve 까지의 역할을 맡는다.

### 파일의 분리

redux/redux-saga의 관심사는 하나의 task 안에서 나뉘며 그 보다 큰 범위의 도메인의 관심사로 분리 할 수 없다. 그러므로 redux/redux-saga는 큰 도메인 아래 saga, redux 각 파일을 생성한다. 예로

```
- user
  - type.js
  - action.js
  - reducer.js
  - saga.js
```

 와 같이 작성한다. 참고로 실제 프로젝트에서 redux관련 파일 작성은 [redux ducks pattern](https://github.com/erikras/ducks-modular-redux)을 사용했다.

### 동기/비동기 이펙트(effect)로 만드는 패턴

## take/call pattern

```javascript
function* workSaga() {
  ...
}

function* watchSaga() {
  while(true) {
    const actions = yield take('EVENT_TYPE');
    yield call(workSaga, actions);
  }
}
```

사가의 가장 기본 패턴이라고 생각하는 코드이다. watch pattern이라고 볼 수 있는데 특정 액션이 발생하면 해당 액션 타입 을 **take** 하고 **call**로 event handler 역할을 하는 saga를 부른다. **call**로 다른 제네레이터 함수를 부르면 불린 제네레이터, 즉 서브루틴은 그 루틴이 모두 끝나야 부모 제네레이터는 next를 호출 가능하다.

***이 패턴을 사용해야 할 때와 사용하지 말아야 할 때:***

앞서 설명했듯이 **call**을 이용해 다른 제네레이터 함수를 부르기 때문에 work 함수가 일을 다 끝내기 전까지는 saga 함수는 액션 타입을 받을 수 없기 때문에 일련의 flow가 모두 끝나기 전에 동일한 flow를 시작하지 않기를 원할때 해당 패턴을 사용하면 된다. 반대로 여러번 요청이 와도 모두 처리해야 한다면 이 패턴을 사용해선 안된다.

- 사용 가능 예:
  - 사용자가 완료 버튼을 누를 때
  - 닫기 버튼을 여러번 누를 때
- 사용 불가능 예:
  - 다른 사가와 연계 역할을 하는 사가

## take/fork pattern

```javascript
function* workSaga1() {
  ...
}
function* workSaga2() {
  ...
}


function* watchSaga() {
  while(true) {
    const actions = yield take('EVENT_TYPE');
    yield fork(work1, actions);
    yield call(work2, actions); // block 발생
  }
}
```

이 패턴은 `take/call pattern`을 조금 변경하여 모든 이벤트를 watch 하고 싶을 때 만들법한 코드이다. 실제 프로젝트에서 이 패턴은 사용하고 있지 않지만 이처럼 사가를 만들었을 때 발생하는 실수에 대해서 설명해 보겠다.

`take/call pattern` 과 다르게 처음 work1을 비동기 이펙트 fork로 호출하고 있다. while 문 안에 ***take*** 이후의 모든 코드가 비동기 라면 해당 패턴은 원하는 대로 동작하겠지만 fork이후 block을 발생시키는 어떠한 코드가 있더라도 연속된 이벤트 호출을 무시할 수 있다.
 그 이유는 watchSaga가 이벤트를 받을 수 있는 상태는 generator 함수의 next 호출의 단계가 `yield take`와 맞아 떨어질 때 뿐이기 때문이다.

즉, 이벤트가 발생해도 `yield call` 의 단계에 있다면 해당 이벤트는 무시 될 것이다. 따라서 위 패턴으로 모든 이벤트를 watch하기 보다 다음에 소개할 패턴으로 작성할 것을 권장한다.

이에 대한 자세한 설명은 공식 문서의 [Non-blocking calls](https://redux-saga.js.org/docs/advanced/NonBlockingCalls.html)에서 확인 가능하다.

***이 패턴은 가급적 사용하지 말아야 할 패턴이다***

## takeEvery를 활용한 watch Pattern

```javascript
function* workSaga1() {
  ...
}

function* watchRequestStayRegion() {
  try {
    yield takeEvery('EVENT_TYPE', workSaga1);
  } catch (e) {
    // handle error
  }
}
```

위 패턴은 발생하는 모든 ***EVENT_TYPE*** 요청을 watch 가능한 패턴이다. 그리고 해당 flow의 시작이니 혹시 처리되지 않은 error를 상위의 rootSaga로 전달하지
않기 위해 try/catch를 작성하였다.

- 사용 가능 예:
  - 다른 사가와 연계 역할을 하는 사가
- 사용 불가능 예:
  - 사용자가 완료 버튼을 누를 때
  - 닫기 버튼을 여러번 누를 때

> 지금까지 watchSaga에서 사용할 수 있는 3가지 패턴을 살펴봤다. 이중 `take/fork pattern` 을 제외한 두개의 패턴은 대부분의 watchSaga에서 쓰였으니 상황에 따라 선택하길 바란다.

## flow/work 패턴

세부적인 사가의 구성 패턴에 대해 알아보는 것은 잠시 미루고 어떻게 하면 재용활 가능한 사가를 작성할 수 있을지를 고민해보자. 우선 재활용 가능한 코드를 만들기 위해서는 관심사의 분리가 필요하다. 어플리케이션을 만들때 프로그래밍의 오류와 기획의 오류가 있다. 이를 기획의 상태 있음/없음으로 나누는 큰틀을 만들어 몇가지 룰을 만들었다.

- 기획의 상태가 있는 saga를 ***FLOW SAGA*** 라고 한다.
- 기획의 상태가 없고 리덕스 액션과 일치하는 하나의 기능을 수행할 때 ***WORK SAGA*** 라고 한다.
- flow saga는 work saga들의 모임이다.
- work saga 안에 다른 work saga를 실행하지 않는다.
- 가급적 flow saga 안에 다른 flow saga를 실행하지 않는다.

## Work Saga의 작성법

프로젝트를 진행하면서 대부분의 work saga는 api 관련 처리였다. rest api는 stateless 이므로 생각하는 work saga의 기준과 맞아 떨어졌기에 그랬던것 같다.

다음은 기본 Work Saga의 형태이다.

```javascript
function* workApi(action) {
  try {
    const { payload } = action;
    const [status, { data, msgCode, msg }] = yield call(api, payload));
    // http status error handler
    yield put(success(data));
  } catch (e) {
    yield put(failure(e));
  }
}
```

함수는 인풋/아웃풋으로 외부와 통신하는 것과 비슷하게 사가는 성공/실패 여부의 Redux-Action을 호출 하고 끝을 낸다. 위 코드에서 한가지 생각해 볼 것은 http status의 에러처리이다. 이를 work saga에 추가한 것은 api 실패 여부는 기획적인 요소가 아니라고 판단하였기에 이에따라 실패/성공 처리 로직을 추가 하였다.

위 saga는 http status 에러핸들 이외에 어떠한 처리도 하지 않았기에 기획과 독립 되었다. 따라서 기획적인 요소를 작성할 때 쉽게 가져다 쓸 수 있다.

에러 처리를 할 때 만약

```javascript
function* workApi(action) {
  try {
    const { payload } = action;
    const [status, { data, msgCode, msg }] = yield call(api, payload));
    // http status error handler
    yield put(Actions.success(data));
  } catch (e) {
    throw Error('error!!');
  }
}
```

위 코드 처럼 throw error를 이용한다면 어떻게 될까?

```javascript
yield put(Actions.failure(e));
```

과 비교해 별 다를 것이 없어 보이지만 throw를 이용해 에러처리를 하면 다음과 같은 코드를 작성하게 된다.

- throw error를 받는 한수의 catch 문에 로직이 추가된다.
- throw error를 어디까지 전파 할지 알 수 없다.
- 이미 프로그램적으로 애러처리를 했기 때문에 해당 work의 에러처리를 flow saga에서 하기 어렵다.

이와 같은 이유로 catch문에 throw를 하지 않고 redux-action을 이용하여 처리만 하고 종료하는 것을 기본 패턴으로 사용한다.

## Flow Saga의 작성법

### work saga를 동기적으로 다루는 flow

```javascript
function* flowProfile(actions) {
  try {
    const { payload } = actions;
    const requestProfileTask = yield fork(workProfile, { payload });
    const requestProfileAction = yield take(['SUCCESS_PROFILE', 'FAILURE_PROFILE']);
    if (requestProfileAction.type === 'FAILURE_PROFILE') {
      yield cancel(requestProfileTask);
      yield put(failureProfileFlow());
      return;
    }

    yield put(successProfileFlow());
  } catch (e) {
    yield put(failureProfileFlow());
  }
}
```

코드를 살펴 보면 call 이 아닌 fork로 work saga를 호출 했기 때문에 그 다음 라인에서 ***take*** 이펙트를 이용해 work saga의 성공/실패를 알 수 있다.
이를 바탕으로 work saga에서 처리되지 않은 기획적인 에러 처리를 할 수 있다. 예를 들어 팝업을 보여준다거나 뒤로 간다거나 하는 것들을 처리 가능하다. 게다가 fork를 이용하여 호출 했기 때문에 ***cancel*** 이펙트를 사용해 제너레이터 함수를 취소 할 수도 있다.

### 다수의 work saga를 병렬처리 하는 flow

flow saga를 작성하다 보면

- 여러개의 work를 병렬 처리 하고 싶다.
- 하나의 work라도 실패하면 모든 work를 취소하고 싶다.
- 여러개의 work들이 모두 성공/실패로 완료 되었는지 알고 싶다.

의 조건을 실행하는 기능이 필요하다.

Promise.all의 사용 처럼 여러개의 task를 비동기로 요청하여 결과만 기다리게 하는 flow를 작성해보자.

```javascript
export function* executeParallelTasks(tasks, successActionTypes, failureActionTypes) {
  let _limit = tasks.length;

  while (_limit-- > 0) {
    const actions = yield take([...successActionTypes, ...failureActionTypes]);
    if (failureActionTypes.some(_type => _type === actions.type)) {
      for (const task of tasks) {
        yield cancel(task);
      }
      break;
    }
  }

  return _limit === -1;
}


function* flowSaga(action) {
  try {
    const { payload } = action;
    const workSaga1Task = yield fork(workSaga1, { payload });
    const workSaga2Task = yield fork(workSaga2, { payload });

    const parallelTasks = [workSaga1Task, workSaga2Task];
    const isDoneParallelTasks = yield call(executeParallelTasks, parallelTasks, [SUCCESS_1, SUCCESS_2], [FAILURE_1, FAILURE_2]);

    if (!isDoneParallelTasks) {
      yield put(flowFailure());
      return;
    }

    yield put(flowSuccess());
  } catch (e) {
    yield put(flowFailure(e));
  }
}
```

우선 원하는 workSaga 들을 fork를 활용해 비동기로 호출하고 이를 처리할 동기코드를 ***executeParallelTasks*** 처럼 작성한다. 이를 통해 여러개의 병렬처리 중 하나라도 실패했을 때 처리중인 task를 모두 취소 및 중단 가능하다. 또한 여러개의 work들이 모두 성공/실패로 완료 되었는지도 알 수 있다.

## 결론

지금까지 몇가지의 saga 작성법을 알아보았다. 몇가지 되지 않지만 ***동일한 기준*** 으로 프로젝트의 모든 saga를 작성할 수 있었다. 프로젝트를 처음 시작할 때는 너무 많은 보일러 플레이트를 작성하는 것이 아닌가 하는 걱정이 있었다. 기존에는 react component의 라이프 사이클을 이용해서 api 콜과 후속 처리를 했는데 이것 보다 작업이 느리다고 생각했다. 하지만 재활용 가능한 더 많은 코드를 만들고, 다른 작업자도 따를 수 있는 동일한 원칙을 만들며, react/redux/redux-saga의 관심사를 분리해서 더 견고한 어플리케이션을 만들고 싶다는 욕심에 계속 작업을 이어갔다. 지금에 와서는 전에 작성한 많은 사가를 이용할 수 있어서 편하고 잘했다는 생각이 든다.

## 번외: Saga를 쉽게 재활용 하기위한 redux action의 payload 작성법

> 어떻게 redux action의 payload를 작성해야 미리 작성해둔 saga함수를 쉽게 재활용 할 수 있을까?

일정 기간의 숙박업장을 요청할 때 requestStays 액션을 호출 한다고 생각해보자.

쉽게 생각할 수 있는것은 사용자로 부터 ***startDate, endDate*** 를 받아서 requestStays(startDate, endDate)를 요청하고 이를 사가에서 받아서 requestStaysApi(startDate, endDate)를 호출하는 방식을 예로 들 수 있다.

그렇다면 다른 화면에서도 requestStays를 요청하고 싶다면 어떻게 해야할까? 그리고 사용자로 부터 ***startDate, endDate*** 를 받을 수 없다면 어떻게 해야할까? 아마도 이전에 작성한 flow saga안에서 startDate, endDate를 redux-store에 저장하는 로직을 넣어야 할 것이다.

그러면

```
requestStays(startDate, endDate)
-> save(startDate, endDate)
-> requestStaysApi(startDate || storesStartDate, endDate || storesEndDate)
```

와 같은 플로우가 될 것이다.

이처럼 request action을 취하면 재활용하기에 많은 어려움이 있다.

- 매번 요청에 필요한 안 써도 될지 모를 (startDate, endDate) 와 같은 인자가 필요하다.
- saga에서 api를 요청할 때 참고해야할 인자들이 request의 시작에서 보낸 인자인지 redux-store에 있는 인자인지 기준을 알기 어렵다.

이를 해결하여 requestStays를 만들어 보면 우선

```
save(startDate, endDate)
-> requestStays()
-> requestStaysApi(storesStartDate, storesEndDate)
```

(requestStays -> requestStaysApi) 이 단계가 saga가 책임질 단계이기때문에 이부분만 flowRequest = (requestStays -> requestStaysApi) 라고 정의 하고
필요할때 flowRequest() 만 호출 가능하다. 또한 찹조해야할 인자가 redux-store 안에 있어서 참조 대상이 명확하고 단순하다.

결론을 내보자면 가능한 redux-action의 payload에 적은 인자를 넣으면 더 사용하기 쉽고 단순한 Saga를 작성할 수 있다.

---

### reference

- https://www.freecodecamp.org/news/redux-saga-common-patterns-48437892e11c/
