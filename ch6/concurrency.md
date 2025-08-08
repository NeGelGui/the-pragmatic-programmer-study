# 동시성

## 동시성이란?

둘 이상의 코드 조각이 실행될 때 동시에 실행 중인 것처럼 행동하는 것이다.  

이때, **시간적 결합**이란 개념이 중요하다.  
이는 당면한 문제 해결에 꼭 필요하지 않은 처리 순서를 코드가 강제할 때 생긴다.  
코드 간의 의존성이 코드의 결합을 유발해 변경 가능성을 해치는 것처럼, 시간적 결합 역시 코드의 결합을 유발한다.  

이러한 시간적 결합을 깨뜨리고 동시성을 달성해야한다.  

## 시간적 결합 깨뜨리기

우리가 시간에서 신경써야 할 측면은 두 가지가 있다.  
바로 **동시성**과 **순서**이다.  

우리는 동시에 일어나도 되는 것이 무엇이고, 반드시 순서대로 일어나야 하는 건 어떤 것인지 찾아내야 한다.  
한 가지 방법이 "활동 다이어그램" 그리기다.  
일종의 Directed Graph 를 그려서, 작업들간의 순서를 정의하고, 어떤 일들을 동시에 작업할 수 있을지 기회를 찾아야 한다.  

그렇게 함으로써 불필요한 시간적 결합을 제거할 수 있다.  

## 공유 상태는 틀린 상태

여러 작업들이 공유하는 상태는 버그를 유발한다.  
세마포어나 뮤텍스 등 동시성 제어 기법들을 쓰더라도, 누군가가 깜빡하고 공유 상태에 함부로 접근한다면 다시 문제가 발생한다.  

이럴 때는 API를 바꿔서 공유 상태에 대한 제어를 중앙으로 집중시켜야 한다.  
API의 호출자는 동시성 제어를 몰라도 되도록 해야 한다.  

이러한 노력에도 불구하고, 리소스를 공유하는 환경에서 동시성은 어렵다.  

아래 농담을 음미해보고, 다음 주제로 넘어간다.

```text
의사 선생님, 이렇게 하면 아파요.
그러면 그렇게 하지 마세요.
```

## 액터와 프로세스

아래와 같은 개념을 정의한다.

- **액터**
  - 자신만의 비공개 지역 상태를 가진 독립적인 가상 처리 장치(virtual processor)
  - 어떤 "우편함"을 가지고 있고, 우편함에 메시지가 도착하면 메시지를 처리한다.
  - 메시지를 처리할 때 다른 액터를 생성하거나, 알고 있는 다른 액터에게 메시지를 보내거나, 다음 메시지를 처리할 때의 상태가 될 새로운 상태를 생성할 수 있다.

이러한 액터는 다음과 같은 특징을 가진다.

1. 액터를 중앙에서 관리하는 것이 하나도 없다.
2. 시스템이 저장하는 상태는 오직 메시지 그리고 각 액터의 지역 상태뿐이다. 이 지역 상태들은 밖에서는 접근이 불가능하다.
3. 모든 메시지는 일방향이다. 답장이란 개념은 없다. 답장을 받으려면 메시지를 보낼 때 답장 받을 우편함 주소를 메시지에 포함해서 보내야 한다. 이 주소로 보내는 답장도 결국 메시지일 뿐이다.
4. 액터는 한 번에 하나의 메시지만 처리한다.

따라서, 액터는 공유 상태 없이 동시성을 구현한다.  

아래가 액터 구현의 예시이다.  

레스토랑을 시뮬레이션한 것으로,  
고객은 파이를 주문하고, 종업원이 주문을 처리하고 파이를 서빙하며, 진열장은 파이의 재고를 관리한다.  

출처: <https://media.pragprog.com/titles/tpp20/code/concurrency/actors/index.js>

```javascript
/***
 * Excerpted from "The Pragmatic Programmer, 20th Anniversary Edition",
 * published by The Pragmatic Bookshelf.
 * Copyrights apply to this code. It may not be used to create training material,
 * courses, books, articles, and the like. Contact us if you are in doubt.
 * We make no guarantees that this code is fit for any purpose.
 * Visit http://www.pragmaticprogrammer.com/titles/tpp20 for more book information.
***/
const { start, dispatch, stop, spawn, spawnStateless } = require('nact');

const router = (module, state, msg, context) => {
  let action = module[msg.type];
  if (action && typeof (action) == "function") {
    const nextState = action(msg, context, state);
    return nextState !== undefined ? nextState : state;
  }
  else {
    console.log(`${context.name} customer ignores unknown message:`, msg);
    return state;
  }
}

const start_actor = (actors, name, module, initial_state={}) => {
    return spawn(actors,
                 (state, msg, context) => {
                   return router(module, state, msg, context)
                 },
                 name,
                 { initialState: initial_state }
    );
}


const sleep = (milliseconds) => {
  return new Promise(resolve => setTimeout(resolve, milliseconds))
}

////////////////////////////////////////////////////////////////////////////////////////

const pieCaseActor = {
  'get slice': (msg, context, state) => {
    if (state.slices.length == 0) {
      dispatch(msg.waiter,
               { type: 'error', msg: "no pie left", customer: msg.customer })
      return state
    }
    else {
      var slice = state.slices.shift() + " pie slice";
      dispatch(msg.customer,
               { type: 'put on table', food: slice });
      dispatch(msg.waiter,
               { type: 'add to order', food: slice, customer: msg.customer });
      return state;
    }
  }
}

const waiterActor = {
  "order": (msg, ctx, state) => {
    if (msg.wants == "pie") {
      dispatch(state.pieCase,
               { type: "get slice", customer: msg.customer, waiter: ctx.self })
    }
    else {
      console.dir(`Don't know how to order ${msg.wants}`);
    }
  },

  "add to order": (msg, ctx) =>
    console.log(`Waiter adds ${msg.food} to ${msg.customer.name}'s order`),

  "error": (msg, ctx) => {
    dispatch(msg.customer, { type: 'no pie left', msg: msg.msg });
    console.log(`\nThe waiter apologizes to ${msg.customer.name}: ${msg.msg}`)
  }

};

const customerActor = {
  'hungry for pie': (msg, ctx, state) => {
    return dispatch(state.waiter,
                    { type: "order", customer: ctx.self, wants: 'pie' })
  },

  'put on table': (msg, ctx, _state) =>
    console.log(`${ctx.self.name} sees "${msg.food}" appear on the table`),

  'no pie left': (_msg, ctx, _state) =>
    console.log(`${ctx.self.name} sulks��`)
}

/////////////////////////////////////////////////////////////////////////////////

const actorSystem = start();

let pieCase = start_actor(
  actorSystem,
  'pie-case',
  pieCaseActor,
  { slices: ["apple", "peach", "cherry"] });

let waiter = start_actor(
  actorSystem,
  'waiter',
  waiterActor,
  { pieCase: pieCase });

let c1 = start_actor(actorSystem,   'customer1',
                     customerActor, { waiter: waiter });
let c2 = start_actor(actorSystem,   'customer2',
                     customerActor, { waiter: waiter });

dispatch(c1, { type: 'hungry for pie', waiter: waiter });
dispatch(c2, { type: 'hungry for pie', waiter: waiter });
dispatch(c1, { type: 'hungry for pie', waiter: waiter });
dispatch(c2, { type: 'hungry for pie', waiter: waiter });
dispatch(c1, { type: 'hungry for pie', waiter: waiter });
sleep(500)
  .then(() => {
    stop(actorSystem);
  })
```
