# ts-context-api-tutorial

[velpert](https://velog.io/@velopert)님의 [TypeScript 환경에서 리액트 Context API 제대로 활용하기](http://velog.io/@velopert/typescript-context-api)실습

# 프로젝트 준비

```bash
$ npx create-react-app ts-context-api-tutorial --typescript
```

- **TodoForm.tsx:** 새 할 일을 등록할 때 사용
- **TodoItem.tsx:** 할 일에 대한 정보를 보여줌
- **TodoList.tsx:** 여러 TodoItem들을 렌더링해줌

# Context 준비하기

만약 상태와 디스패치 함수를 한 Context 에 넣게 된다면, TodoForm 컴포넌트처럼 상태는 필요하지 않고 디스패치 함수만 필요한 컴포넌트도 상태가 업데이트 될 때 리렌더링하게 됩니다. 두 개의 Context를 만들어서 관리한다면 이를 방지 할 수 있습니다.

## 상태전용 Context 만들기

**src/contexts/TodosContext.tsx**

```tsx
import { createContext } from "react";

// 나중에 다른 컴포넌트에서 타입을 불러와서 쓸 수 있도록 내보내겠습니다.
export type Todo = {
  id: number;
  text: string;
  done: boolean;
};

type TodosState = Todo[];

const TodosStateContext = createContext<TodosState | undefined>(undefined);
```

Context 를 만들땐 위 코드와 같이 `createContext` 함수의 Generics 를 사용하여 Context에서 관리 할 값의 상태를 설정해줄 수 있는데요, 우리가 추후 Provider를 사용하지 않았을 때에는 Context의 값이 `undefined` 가 되어야 하므로, `<TodosState | undefined>` 와 같이 Context 의 값이 `TodosState` 일 수도 있고 `undefined` 일 수도 있다고 선언을 해주세요.

## 액션을 위한 타입 선언하기

- **CREATE:** 새로운 항목 생성
- **TOGGLE:** done 값 반전
- **REMOVE:** 항목 제거

**src/contexts/TodosContext.tsx**

```tsx
import { createContext, Dispatch } from "react";

export type Todo = {
  id: number;
  text: string;
  done: boolean;
};

type TodosState = Todo[];

const TodosStateContext = createContext<TodosState | undefined>(undefined);

type Action =
  | { type: "CREATE"; text: string }
  | { type: "TOGGLE"; id: number }
  | { type: "REMOVE"; id: number };

type TodosDispatch = Dispatch<Action>;
const TodosDispatchContext = createContext<TodosDispatch | undefined>(
  undefined
);
```

이렇게 액션들의 타입을 선언해주고 나면, 우리가 디스패치를 위한 Context를 만들 때 디스패치 함수의 타입을 설정 할 수 있게 됩니다.

이렇게 `Dispatch` 를 리액트 패키지에서 불러와서 Generic으로 액션들의 타입을 넣어주면 추후 컴포넌트에서 액션을 디스패치 할 때 액션들에 대한 타입을 검사 할 수 있습니다. 예를 들어서, 액션에 추가적으로 필요한 값 (예: `text`, `id`)이 빠지면 오류가 발생하죠.

## 리듀서 작성하기

**src/contexts/TodosContext.tsx**

```tsx

(...) // 이전 코드 생략

function todosReducer(state: TodosState, action: Action): TodosState {
  switch (action.type) {
    case "CREATE":
      const nextId = Math.max(...state.map(todo => todo.id)) + 1;
      return state.concat({
        id: nextId,
        text: action.text,
        done: false
      });
    case "TOGGLE":
      return state.map(todo =>
        todo.id === action.id ? { ...todo, done: !todo.done } : todo
      );
    case "REMOVE":
      return state.filter(todo => todo.id !== action.id);
    default:
      throw new Error("Unhandled action");
  }
}
```

## TodosProvider 만들기

```tsx
import React, { createContext, Dispatch, useReducer } from 'react';

(...) // (이전 코드 생략)

export function TodosContextProvider({ children }: { children: React.ReactNode }) {
  const [todos, dispatch] = useReducer(todosReducer, [
    {
      id: 1,
      text: 'Context API 배우기',
      done: true
    },
    {
      id: 2,
      text: 'TypeScript 배우기',
      done: true
    },
    {
      id: 3,
      text: 'TypeScript 와 Context API 함께 사용하기',
      done: false
    }
  ]);

  return (
    <TodosDispatchContext.Provider value={dispatch}>
      <TodosStateContext.Provider value={todos}>
        {children}
      </TodosStateContext.Provider>
    </TodosDispatchContext.Provider>
  );
}
```

## 커스텀 Hooks 두 개 작성하기

우리가 추후 TodosStateContext와 TodosDispatchContext를 사용하게 될 때는 다음과 같이 `useContext`를 사용해서 Context 안의 값을 사용 할 수 있습니다.

```tsx
const todos = useContext(TodosStateContext);
```

그런데 이 때 `todos` 의 타입은 `TodosState | undefined` 일 수 도 있습니다. 따라서, 해당 배열을 사용하기 전에 꼭 해당 값이 유효한지 확인을 해주어야 하죠.

```tsx
const todos = useContext(TodosStateContext);
if (!todo) return null;
```

그렇게 해도 상관은 없지만.. 더 좋은 방법은 TodosContext 전용 Hooks를 작성해서 조금 더 편리하게 사용하는 것 입니다. 다음과 같이 코드를 작성하면 추후 상태 또는 디스패치를 더욱 편하게 이용 할 수도 있고, 컴포너트에서 사용 할 때 유효성 검사도 생략 할 수 있습니다.

**src/contexts/TodosContext.tsx**

```tsx
import React, { createContext, Dispatch, useReducer, useContext } from 'react';

(...)

export function useTodosState() {
  const state = useContext(TodosStateContext);
  if (!state) throw new Error('TodosProvider not found');
  return state;
}

export function useTodosDispatch() {
  const dispatch = useContext(TodosDispatchContext);
  if (!dispatch) throw new Error('TodosProvider not found');
  return dispatch;
}
```

이렇게 만약 함수 내부에서 필요한 값이 유효하지 않다면 에러를 `throw` 하여 각 Hook이 반환하는 값의 타입은 언제나 유효하다는 것을 보장 받을 수 있습니다. (만약 유효하지 않다면 브라우저의 콘솔에 에러가 바로 나타납니다.)

# 컴포넌트에서 Context 사용하기

## TodosContextProvider로 감싸기

```tsx
import React from "react";
import TodoForm from "./components/TodoForm";
import TodoList from "./components/TodoList";
import { TodosContextProvider } from "./contexts/TodosContext";

const App = () => {
  return (
    <TodosContextProvider>
      <TodoForm />
      <TodoList />
    </TodosContextProvider>
  );
};

export default App;
```

## TodoList 에서 상태 조회하기

**src/components/TodoList.tsx**

```tsx
import React from "react";
import TodoItem from "./TodoItem";
import { useTodosState } from "../contexts/TodosContext";

function TodoList() {
  const todos = useTodosState();
  return (
    <ul>
      {todos.map(todo => (
        <TodoItem todo={todo} key={todo.id} />
      ))}
    </ul>
  );
}

export default TodoList;
```

그냥 `useTodosState` 를 불러와서 호출하기만 하면 현재 상태를 조회 할 수 있답니다.

## TodoForm 에서 새 항목 등록하기

이제 TodoForm 에서 새 항목을 등록하는 작업을 구현해봅시다. `useTodosDispatch` Hook 을 통해 `dispatch` 함수를 받아오고, 액션을 디스패치해보세요.

**src/components/TodoForm.tsx**

## TodoItem 에서 항목 토글 및 제거하기

아마 일반적으로 Context를 사용하지 않는 환경에서는 TodoItem 컴포넌트에서 `onToggle`, `onRemove` props 를 가져와서 이를 호출하는 형태로 구현 할 것 입니다. 하지만 지금과 같이 Context를 사용하고 있다면 굳이 이 함수들을 props를 통해서 가져오지 않고, 이 컴포넌트 내부에서 바로 액션을 디스패치하는 것도 꽤나 괜찮은 방법입니다.

**src/components/TodoItem.tsx**

# 정리

이번 튜토리얼에서는 우리가 타입스크립트 환경에서 리액트의 Context API를 효과적으로 활용하는 방법에 대해서 알아보았습니다. Context API를 사용하여 상태를 관리 할 때 useReducer를 사용하고 상태 전용 Context 와 디스패치 함수 전용 Context를 만들면 매우 유용합니다. 추가적으로, Context를 만들고 나서 해당 Context를 쉽게 사용 할 수 있는 커스텀 Hooks를 작성하면 더욱 편하게 개발을 하실 수 있습니다.

이 튜토리얼에서 사용하는 방식은 꽤나 편한 방식이지만 무조건 정답은 아닙니다.
