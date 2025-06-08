---
title: "TypeScriptでユニークな配列か型チェックした"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "contest2025ts"]
published: true
---

## 始めに

`Array<'A' | 'B' | 'C'>`と型定義した場合、`A`か`B`か`C`のどれかを使った配列であることを制約づけることは可能ですが、`A`が複数個設定できてしまい、ユニークな配列として初期化することはできません。

以前`['A', 'B', 'C']`のようなタプル型にしたら実現できるのでは？と思い、以下のような型定義をしたことがあります。

```ts:全パターンのタプルに変換する型
type UniqueArray<T, U = T> = [T] extends [never]
  ? []
  : T extends any
    ? [T, ...UniqueArray<Exclude<U, T>>] | UniqueArray<Exclude<U, T>>
    : never
```

これを使って例えば`UniqueArray<'A' | 'B' | 'C'>`と定義するとスクショのような型が出てきて、何を渡したい型か良く分からなくなります😂 この状況なのでエラーになった時も何が問題かパッと見じゃ良く分からないと思います。

![](/images/type-check-unique-array/type-unique-array.png)

![](/images/type-check-unique-array/type-unique-array-error.png)

検証コードをTypeScript Playgroundに書きましたので興味があるかたはこちらをご参照ください。

https://www.typescriptlang.org/play/?#code/C4TwDgpgBAqgdgSwI4FcIEEBOmCGIA8AKgDSxQC8UhAfBVANqEC6UEAHsBHACYDODcCADcImJgCgoUAPwMJUgFxVWHLnyg44ISVJkMSUAHTH4yNFlwEAomwDGAGxTcI+GKRrUWAH1iJUGbDx8GwcnFzcqamodRShBEUxxcVsAezheYA1sJVN-CyCAcnQCqB8CgCESsoBhAtpKegLa0iaCluKJIA

更に以下のような記事を見かけ、TypeScriptには向き不向きがあることを実感して、初期代入時くらいは気をつけるかどうしても気になる場合はユニーク処理を通すようにしたら良いだけなので諦めることにしました。

https://zenn.dev/dqn/articles/union-to-tuple

しかし今回Zennから「TypeScriptでやってみた挑戦・学び・工夫」というテーマの記事投稿コンテストが出たことで、改めてユニークな配列をチェックできないかChatGPTにも聞いて挑戦することにしました💪

https://zenn.dev/contests/contest2025ts

最終的にはメソッド経由であればユニークなデータで宣言しているかをチェックすることはできたのでそれについてまとめたいと思います。

## そもそもユニークな配列であるかチェックする必要がないケース

ユニークな配列をチェックする方法について話す前に、そもそも実装次第ではそれをやる必要ないケースもあると思います。TypeScriptは向き不向きがあるため、別なアプローチを取ることで簡単に実装できる場合があるのでまずはそのケースに当てはまっていないか考えても良いと思いました。パッと思いついたのは以下のようなケースです。

### 全ての項目を宣言させたいケース

最初の例で言うと`['A', 'B', 'C']`と全部の項目を入れたい場合です。この場合は型を定義してから宣言するより、先に配列を作ってからそれを元に型を作った方が圧倒的に楽です。

```ts:先に全パターンの配列を作って、そこから型を作る
const TYPES = ['A', 'B', 'C'] as const

type Type = typeof TYPES[number]
type TypeArray = Type[]
```

### 順番は関係なく、使う/使わないだけ指定したいケース

`['A', 'B', 'C']`という順番は決まっていて、そこから`A`と`C`だけ使いたいみたいなケースの場合は、booleanオブジェクトで使う/使わないの情報を渡すようにするとTypeScript的には楽になります。全部書くのが面倒であれば`Partial`を使うことで必要な項目だけ指定することができます。

```ts:順番は関係ない場合は使う/使わないをフラグで渡す
const TYPES = ['A', 'B', 'C'] as const
type Type = typeof TYPES[number]

const func = (useTypeMap: Partial<Record<Type, boolean>>) => {
  // 使用するtypeのみ抽出する
  const usableTypes = TYPES.filter((type) => useTypeMap[type])
}

func({
  A: true,
  C: true,
})
```

## ユニークな配列で初期化しているかをTypeScriptでチェックする方法

それでは本題に入りたいと思います。TypeScriptではunion型だとそれぞれの型をチェックするのは難しいですが、タプル型であれば一つずつ見ることができるので、そこから重複されている項目を見つけてエラーにすることができましたのでその方法で実装します。

### タプルから重複項目を抽出する

まずはタプルから重複項目を抽出します。コードは以下のようになり、処理の流れは簡単に説明すると以下のようになります。

- 最初に`Head`と`Rest`でタプルの先頭と残りに分割
  - `Head`が今までにみた`Seen`という配列の全パターンのいずれかに含まれているかチェック
    - 含まれている場合は重複なので第3引数(`Duplication`)に`Head`を追加
    - 含まれていない場合は新規なので第2引数(`Seen`)に`Head`を追加
  - `Rest`で再帰呼び出しする
- 分割不可能な状態になったら（全項目をチェックしたら）`Duplication`を返す

```ts:タプルから重複項目を抽出する型
type DuplicationKeys<
  Arr extends readonly any[],
  Seen extends readonly any[] = [],
  Duplication extends readonly any[] = []
> =
  Arr extends readonly [infer Head, ...infer Rest]
    ? Head extends Seen[number]
      ? DuplicationKeys<Rest, Seen, [...Duplication, Head]>
      : DuplicationKeys<Rest, [...Seen, Head], Duplication>
    : Duplication
```

これを実行すると以下のような型が出力されます。重複なしの場合は空配列、重複ありの場合はその項目が出ます。

```ts
type NoDuplicationKeys = DuplicationKeys<['A', 'B', 'C']> // = []
type HasDuplicationKeys = DuplicationKeys<['A', 'B', 'B']> // = ['B']
```

### ユニークな配列になっているか判定する

続いて`DuplicationKeys`を使ってユニークな配列じゃない場合はエラーになるような形にします。問題ない場合はそのまま返します。エラーにするかの判定は`DuplicationKeys`に対して`[infer Head, ...infer Rest]`をextendsして少なくとも1個は項目があるかで判定しています。エラーにする方法としては`never`を返してしまうのが一番単純ですが、何故`never`になっているのかが分からないので、以下のやり方を参考にエラーメッセージを含んだオブジェクト型を返すようにしました。

https://zenn.dev/dqn/articles/union-to-tuple#%E3%81%8A%E3%81%BE%E3%81%91

```ts:ユニークな配列になっているか判定する型
type CheckUniqueArray<Arr extends readonly any[]> =
  DuplicationKeys<Arr> extends readonly [infer Head, ...infer Rest]
    ? { __duplication: [Head, ...Rest] }
    : Arr
```

これを実行すると以下のような型が出力されます。

```ts
type OkUniqueArray = CheckUniqueArray<['A', 'B', 'C']> // = ['A', 'B', 'C']
type NgUniqueArray = CheckUniqueArray<['A', 'B', 'B']> // = { __duplication: ['B'] }
```

### メソッドを経由して型チェックする

`CheckUniqueArray`をジェネリクスのメソッドに通して型チェックさせます。柔軟性を持たせるため項目の型を指定するメソッドを更にラップすると以下のようなコードになります。

```ts:ユニーク配列生成器を生成するメソッド
const uniqueArrayCreator = <AllowType extends any>() => {
  const uniqueArray = <Arr extends readonly AllowType[]>(
    arr: CheckUniqueArray<Arr>
  ) => {
    return arr
  }
  return uniqueArray
}
```

実行すると以下のようになり、重複している場合は重複している項目が表示されてエラーが出るようになりました🎉 また`'A' | 'B' | 'C'`以外の文字を指定した場合もキチンとエラーになっています。

```ts
const uniqueArray = uniqueArrayCreator<'A' | 'B' | 'C'>()

// pass
const arr1 = uniqueArray(['A', 'B', 'C'] as const)

// Argument of type 'readonly ["A", "B", "B"]' is not assignable to parameter of type '{ __duplication: ["B"]; }'.
//  Property '__duplication' is missing in type 'readonly ["A", "B", "B"]' but required in type '{ __duplication: ["B"]; }'.(2345)
const arr2 = uniqueArray(['A', 'B', 'B'] as const)

// Type '"D"' is not assignable to type '"A" | "B" | "C"'.(2322)
const arr3 = uniqueArray(['A', 'B', 'C', 'D'] as const)
```

![](/images/type-check-unique-array/type-error-duplication.png)

![](/images/type-check-unique-array/type-error-undefined-type.png)

### 検証コード

今回書いたコードは以下に書かれていますので、興味がある方はこちらからご参照ください。

https://www.typescriptlang.org/play/?#code/PQKhCgAIUx+hkdYZDXDIaQZCRDIWcTDgkYUAzB3boJIZBeo0C-FQTQZBohihGHABcBPABwFNIARAV0YBsBLAYwCGtXgHsAdgGlm9AM4AeKJACCAJ1WRmAD1rNxAE1mRVzQfond6kQePoBtALoAaJQGVmezTr2Hjp8+KW1raOkAC8kI4ukOxcfEIiEl66BkYmZhZWNvYO4ZEO4AB84UpqGtopvukBQXa84gBmzBoAEv5OkAB03fVNGgBKzLK0BTExAPyQbWbJPkbuenbiHAC2AEbNo2MTsTwCwmJSMgqDwx0L4h123Z2cewmHHdP6DoVK2wBcu-EHEtJy8lOtCuNwuT38zm++0S4jenyhDwk4DoTFYADlRHcfjD-kYIljoYdcfI7AByZSkjqkgBClMgpIAwqTXiiWFNBLICYijnI8lzfjyFGSKVTaaLmW9wKAINBIIAzhkA0wyAH4ZAPUMgCsGQCyiYB0JUA1gxqwDGDIAzBkAIgzkZCAQYZAOUMgGGGFUUKg0BhshkAC2Y-AA1gBVcS8ACOHGYZUE9HkZVmqT8GUCWRCrxKMX5OOOYfUxQqcyjNSsdUazSm7S6PTzAyGI3ekEmAG9IAB9Wv6OKEiRfOzPDo3IG5AC+Fa+ZWRTtYAHlvb6A0H1CG8q73WP-YHg6HhXSaaumSyh5A0QBzH0LyeqacRWee-cTpck8mrsX02ksqVgKjy5Uq7WAfFdAAhGgAs1fBf+3QDQ-ASMMkAcOOi5TvQDLpLQogaBEYbcNwogAO4ACqohGvjZIUAAUACU4TFFWSjAeIoHgQeS55Km5TeJG1SZCoyFoZhLCOPhFaCOoXynvOF5QXRcKQERYQkRWJi0BwqjiNY6hKL2MRSTJclUYJR70OAvbgORlEQYex5gQZS4waYcGqPI16QAAPnepK2fSTL4QRyJ6bQ8mqAAjHk6mQZpeEruKVIbtYRjua57meQATL5JlQYF17BfZuQcpAEW6SBHk8aoADMcXUQlQX2SFq5sMyYXpVlrlAA

## 応用: オブジェクトの特定のキーに対して重複チェックする

先ほどの実装で、`DuplicationKeys`のところを微調整することでオブジェクトの特定のキーに対して重複チェックすることも可能です。
例えば以下のオブジェクトで、`type`というキーの重複チェックをする場合をやってみます。

```ts
type Item<T extends string = string> = {
  type: T
  name: string
}
```

type名も`DuplicationTypes`の方が合っていそうなのでそちらの名前にリネームして、以下のようになります。変更箇所が全部なのでdiff表示の意味があるか微妙なところですが、対比として見やすいように最初のコードも載せています。

```diff ts:typeキーの重複チェックに変更
-type DuplicationKeys<
-  Arr extends readonly any[],
-  Seen extends readonly any[] = [],
-  Duplication extends readonly any[] = []
-> =
-  Arr extends readonly [infer Head, ...infer Rest]
-    ? Head extends Seen[number]
-      ? DuplicationKeys<Rest, Seen, [...Duplication, Head]>
-      : DuplicationKeys<Rest, [...Seen, Head], Duplication>
-    : Duplication

+type DuplicationTypes<
+  Arr extends readonly Item[],
+  Seen extends readonly Item[] = [],
+  Duplication extends readonly Item['type'][] = []
+> =
+  Arr extends readonly [infer Head extends Item, ...infer Rest extends readonly Item[]]
+    ? Head['type'] extends Seen[number]['type']
+      ? DuplicationTypes<Rest, Seen, [...Duplication, Head['type']]>
+      : DuplicationTypes<Rest, [...Seen, Head], Duplication>
+    : Duplication
```

他のコードも微調整して`uniqueArray`に`type`が重複されている配列を渡すとキチンとエラーが出ました。メッセージが長くて読みづらいですが、`{ __duplication: ['B'] }`が出て重複のtypeが分かるようになっています。

![](/images/type-check-unique-array/type-error-duplication-item-type.png)

細かい調整部分の説明はここでは割愛しますので、詳細を見たい方はこちらをご参照ください。

https://www.typescriptlang.org/play/?#code/C4TwDgpgBAksEFsA8AVKEAe8B2ATAzlPsAE4CW2A5lALxGkWUB8tUA3gFBRSiQBcUFFyjYAhgggDi5KhwC+HDgHoAVCq4qogfoZA6wyBrhkDSDIEiGQLOJgcEjAoBmA7t0BJDIF6jQF+KgTQZA0QwalHXtAAiAVzAANmQAxqLAZAD22CjgEPhIwgCCJCToWBB4hCQQorhRASCw8AgA2gC6ADTCAMoQGWk4BFDZufmFcIjlrOVV3H6BIWGR2A0ZTS152AVFnQDkXrNlXXTlHCw0SSmjmc05k9MlFABmEKkAEnvbTR0IFVAAdI-Hp1AASnHAV1l7bTOlZWVhNwAPxQC65ErzWKLL5QWoZErYXwIABGpyWUMgiyB3CgoP6QVC4SiMUg8XexDu8OwdxKj3uBMGxJpYL2kIWAKYOO4AkZROGpLiSApwFp9Opd3BuEqUD5QyiXNxUF5-kJ8uwii8UAAchE5czBYQ6PqBbF4iVhGweLEBLNErM7mIJLatIAShh09qgcl67Gt-CgswAQg6ROJJAG3Tpg16fVavLaAMIhp3h2aRpNejhlLlas6ifAmklm1iF6JmpAW7hxm0B+2OsMu92e72Wv2p4P150R93RltVtu2juhrtpnsAfQATLNM9nFKp1FBNIAzhkA0wyAH4ZAPUMgCsGQCyiYB0JUA1gxbwDGDIAzBkAIgwuAyAQYZAOUMgGGGDeudyeWJQBMACwgwQA1gBVbAyAAR18CBkhIUQQCQcDYQmX4bnKdZhFLQ1oJSFhMEab5WimQpDmwE5zj2O56WeVIRUBJVQStMcx1wVUmWGAQSilEjHgozMlQEcDNTfAB5f9AJAsCUkg1hP2-QTgNA8DIIrVt41rZMG27D1pz7X1FKDZSR0jXtYwHAMk07VN03UrMczfbVKAA6SRIgwo6Ak39bOE2SoMrTSaztHTTKbdSDK0ocU0bKMAoU7zgpU0co0nczZ2UNQNCgVdN33QB8V0ABCNAAs1GwsufRcPGCKJiCgXwhJk0SQATFpgAiVI6GggIAgiAB3QVYWkRgmAACgASloFhOG4YrsFK8q7Pc1g0NSTCxmw-Z2mKJqWva2ImEQnqcVEFIBGcqS3KqmbFSgAaaCGnFsmAXwSBGHaSGEBRuCum6Rgmw6HPkRRRvGir7LEuh3sqhyapyOqSCQHyoAAHwDaNYdmJNer676Ss+e6AEZWCB-6QB6zzq39HyTNC5tAsi3zQv0iKieM4c-J0DMFDKKB8ygH7gBRjnWZSCdsb+9z8Zp1M63p0nwv7ILKdU6nJYpkmZbimdWcIDmubRnmSAAZn5yaqqFuWidFkLVLJ4XB2lmLZa82nLbMmNzYDbw7fdZ3lbZtWOCAA

## 終わりに

以上がTypeScriptでユニークな配列かチェックする方法でした。メソッドの引数に渡した配列が重複であるかを型チェックすることができたため、メソッド経由であればユニークであることが保証できるようになりました。重複時もエラーメッセージを含んだ型に変換することである程度エラー原因の特定がしやすくて実用に堪えられそうな印象でした。しかしこの辺の処理はTypeScriptにとっては苦手な領域であるため（ユニークな配列型って愚直に考えると全パターンのタプルでしか定義できなそう。。）、そもそもユニーク配列をチェックする必要性がない形に実装を変える工夫をまずは考えた方が良いとは思いました。
それでもユニークな配列かを型でチェックしたい場合、こちらの記事が参考になれば幸いです。
