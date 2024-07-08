---
title: "Next.js v14 & React v18.3 互換の useActionState"
emoji: "💥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [next, react, useActionState]
published: true
publication_name: yumemi_inc
---

## 前置き

Next.js で server action を利用する際に、実行中の状態を判定したい機会があり、 `useActionState` を使いたくなったのですが、現時点では、Next.js v14 と React v18 を利用している環境では、まだ利用できないこともあり、それに変わる方法として代替の hooks を実装したのでご紹介します。

:::message
実は、Next v14 利用環境では、`useActionState` の型定義が参照でき、IDE で上では利用できそうに見えるのですが、実装が存在しないというややこしい状態にあったりします 😢
:::

## `useActionState` とは

React 19 RC のリリース記事を見ると、React 19 で、React 18 Canary で追加された `useFormState` から改名したものであり、

> React.useActionState は以前の Canary リリースでは ReactDOM.useFormState と呼ばれていましたが、名前を変更し、useFormState を非推奨にしました。
> 詳細は [#28491](https://github.com/facebook/react/pull/28491) を参照してください。
> https://ja.react.dev/blog/2024/04/25/react-19#new-hook-useactionstate

また、上記に記載の React への PR では、以下のように紹介されており、

> this hook is the automatic tracking of the return value and pending states of the wrapped function.
> 意訳: このフックはラップされた関数の戻り値とペンディング状態を自動的にトラッキングします。

上述のうちの「ペンディング状態のトラッキング」ができる点が、`useFormState` とは異なる点である事がわかります。

## `useFormState` と `useTransition` で再現する

そこで、「ペンディング状態のトラッキング」ができる `useFormState` が実装できれば、`useActionState` の代替になりうると考え、`@types/react-dom/canary.d.ts` を参考に `useActionStateCompat` として、以下のように実装してみました。

```ts
import { useCallback, useTransition } from "react";
import { useFormState } from "react-dom";

/**
 * @see {@link https://github.com/DefinitelyTyped/DefinitelyTyped/blob/0b728411cd1dfb4bd26992bb35a73cf8edaa22e7/types/react/canary.d.ts#L103-L112}
 */
export function useActionStateCompat<State>(
  action: (state: Awaited<State>) => State | Promise<State>,
  initialState: Awaited<State>,
  permalink?: string,
): [state: Awaited<State>, dispatch: () => void, isPending: boolean];
export function useActionStateCompat<State, Payload>(
  action: (state: Awaited<State>, payload: Payload) => State | Promise<State>,
  initialState: Awaited<State>,
  permalink?: string,
): [
  state: Awaited<State>,
  dispatch: (payload: Payload) => void,
  isPending: boolean,
];
export function useActionStateCompat<State, Payload>(
  action: (state: Awaited<State>, payload: Payload) => State | Promise<State>,
  initialState: Awaited<State>,
  permalink?: string,
) {
  const [isPending, startTransition] = useTransition();

  const [currentState, dispatchAction] = useFormState(
    action,
    initialState,
    permalink,
  );

  const finalAction = useCallback(
    (payload: Payload) => {
      startTransition(() => {
        dispatchAction(payload);
      });
    },
    [dispatchAction],
  );

  return [currentState, finalAction, isPending];
}
```

### 使い方

基本的に、`useFormState` と同じ形式で利用でき、期待通り「実行中」の状態を戻り値の配列の3つめの値から参照できます。

```ts

type FormState = { errors: { word?: string } }

const myAction = async (state: FromState, payload: FormData) => {
    "use server";

    const newState = structuredClone(state);

    if (!payload.get('word')) {
        newState.errors.word = '入力してください'
    }

    return newState;
}

function MyForm() => {
  const [currentState, action, isPending] = useActionState(myAction, { errors: {} });

  return (
    <form action={action}>
      <div>
        <input type="text" name="word" />
        {currentState.errors.word && <div>{currentState.errors.word}</div> }
      </div>
      <button disabled={isPending}>Submit</button>
    </form>
  );
}
```

## 結び

React 19 、Next.js 15 がリリースされるころには不要になるとは思いますが、リリース後は `useFormState` が廃止予定となるため、その間に Server Action を積極的に利用しているプロジェクトでは、このようなフックを用意しておくと良いのではないでしょうか？

以下は、`useActionStateCompat` を先に紹介した方法で実装し、パッケージ化したものになります。
プロジェクト毎に用意するのが面倒な場合にご利用ください。

https://github.com/strozw/use-action-state-compat
