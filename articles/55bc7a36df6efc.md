---
title: "Record<string, any>としても実はstring以外のキーも入る"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

## 始めに

以下のようなオブジェクトをフィールドごとに描画方法をカスタマイズして一括で描画できるようにするメソッドを作った際に、Reactのkeyのところで `symbol` は代入できないというエラーが出ました。

```tsx:Record<string, any>なのにkeyof Record<string, any>にsymbolがあると言われる
type RenderItemByKey<Data extends Record<string, any>, K extends keyof Data> = {
  fieldKey: K
  label: string
  render?: (value: Data[K], data: Data) => React.ReactNode
}

type RenderItem<Data extends Record<string, any>> = {
  [K in keyof Data]: RenderItemByKey<Data, K>
}[keyof Data]

const renderDataFields = <Data extends Record<string, any>>(data: Data, items: RenderItem<Data>[]) => {
  return (
    <dl>
      {items.map((item) => {
        const value = data[item.fieldKey]
        const content = item.render ? item.render(value, data) : value
        return (
          // Type 'symbol' is not assignable to type 'Key | null | undefined' というエラーが出る
          <React.Fragment key={item.fieldKey}>
            <dt>{item.label}</dt>
            <dd>{content}</dd>
          </React.Fragment>
        )
      })}
    </dl>
  )
}
```

:::details 使用例

```tsx
type Person = {
  id: number
  name: string
  age: number
  sex: 'mail' | 'femail'
}
const person: Person = {
  id: 0,
  name: '山田太郎',
  age: 20,
  sex: 'mail'
}

const RENDER_ITEMS: RenderItem<Person>[] = [
  { fieldKey: 'id', label: 'ID' },
  { fieldKey: 'name', label: '名前' },
  {
    fieldKey: 'age',
    label: '年齢',
    render: (value) => {
      return `${value}歳`
    }
  },
  {
    fieldKey: 'sex',
    label: '性別',
    render: (value) => {
      return value === 'mail' ? '男性' : '女性'
    }
  }
]
renderDataFields(person, RENDER_ITEMS)
```

:::

![](/images/restrict-string-object/error-symbol-key.png)

とりあえずkeyに渡す部分を`String`でラップすることでstringに変換してくれるのでエラーは解消されますが、なんでこんな挙動になるのか気になったので調査したのでそれをまとめました。

```diff tsx:Stringでラップしてエラーを解消
 const renderDataFields = <Data extends Record<string, any>>(data: Data, items:  RenderItem<Data>[]) => {
   return (
     <dl>
       {items.map((item) => {
         const value = data[item.fieldKey]
         const content = item.render ? item.render(value, data) : value
         return (
-          // Type 'symbol' is not assignable to type 'Key | null | undefined' というエラーが出る
-          <React.Fragment key={item.fieldKey}>
+          // Stringでラップすることでtypeエラーは解消できる
+          <React.Fragment key={String(item.fieldKey)}>
             <dt>{item.label}</dt>
             <dd>{content}</dd>
           </React.Fragment>
         )
       })}
     </dl>
   )
}
```

## `Record<string, any>` としても実はstring以外のキーも入る

かなり衝撃だったのですが、**そもそも`Record<string, any>`としてもsymbolやnumberをキーにしても代入できていました。** この仕様に関するドキュメントを見つけられませんでしたが、なんとなくオブジェクトに`Symbol`や`number`をキーとして入れると自動で`toString`が実行されて`string`として扱われるため、`Record<string, any>`とすると全てのキーが対象になってしまうのかなと思いました🤔

```ts
const sym = Symbol();
// Symbolのオブジェクトを代入できる
const symbolObj: Record<string, any> = { [sym]: 'hoge' }
// numberのオブジェクトも代入できる
const numberObj: Record<string, any> = { [10]: 'hoge' }
```

従って最初に書いていたコードもそもそもシンボルをキーに設定することが可能な状態だったため、シンボルが入ることも考慮した実装にする必要がありました。

```tsx:シンボルをキーにしたオブジェクトも使用できる
const sym = Symbol();

type SymbolObj = {
  [sym]: typeof sym
}
const symObj: SymbolObj = {
  [sym]: sym
}

const RENDER_SYMBOL_OBJ_ITEMS: RenderItem<SymbolObj>[] = [
  {
    // エラーにならず指定できる
    fieldKey: sym,
    label: 'シンボル'
  }
]
renderDataFields(symObj, RENDER_SYMBOL_OBJ_ITEMS);
```

## stringのみに絞る方法

`Record<string, any>` は実はSymbolやnumberもキーにできるようで、実質的に `object` と同じであることが分かりました。今回のユースケースだとSymbolも許容して`String`でラップしても良いかなと思いましたが、どうしてもstringだけに絞りたい時の方法を色々調べてみました。

### fieldKeyをstringだけに絞る

`Record<string, any>` はSymbolやnumberも入るため、 `keyof Record<string, any>` は `string | number | symbol` になります。ここに `string` でインターセクションをかけると `string` のみになるため、これでstringだけに絞りたいと思います。今回の例だとunionを作るところに `& string` を足すことでstringのfieldKeyだけのパターンだけ抽出され、結果的にそれ以外のfieldKeyは設定出来なくなります。

```diff tsx:stringとインターセクションしてstring以外のunionを取り除く
 type RenderItem<Data extends Record<string, any>> = {
   [K in keyof Data]: RenderItemByKey<Data, K>
-}[keyof Data]
+}[keyof Data & string] // keyof の中で更にstringのものだけに絞り込む
```

### ジェネリクスにKeyを追加してそこにstring制約をかける

`Record<string, any>` ではキーをstringに絞れないため、前段に更にジェネリクスを追加して、そこにstring制約を設定することでキーをstringだけに制限することができます。

```diff tsx
-type RenderItem<Data extends Record<string, any>> = {
-  [K in keyof Data]: RenderItemByKey<Data, K>
-}[keyof Data]
+type RenderItem<Key extends string, Data extends Record<Key, any>> = {
+  [K in Key]: RenderItemByKey<Data, K>
+}[Key]
```

これによって固くはなりましたが、引数が一つ増えて定義がちょっと手間になったのと、keyofを使わず期待していないキーを設定することもできるようになり、むしろ抜け道がありそうでちょっと微妙かなと思いました。

```ts
const RENDER_ITEMS: RenderItem<
  // string以外のキーがあるとエラーになる
  keyof Person,
  Person
>[] = [
  ...
]
```

## 終わりに

以上が `Record<string, any>` で絞っているのにキーがsymbolも入っていた理由と対処方法でした。`Record<string, any>`と書いているのにstringで絞れなかったのは衝撃ですが、symbolが入っていても基本的には`String`でラップすれば良いのでそこまで実装に困らなそうだなと思いました。どうしてもstringで絞りたい場合は `& string` とstringでインターションすると良いと分かったため、困ったときはそれで対処しようかなと思いました。

最後に検証コードをTypeScript Playgroundに書きましたので詳細のコードが気になる方はこちらをご参照いただけると幸いです。

https://www.typescriptlang.org/play/?#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-VRtmBA8fJOzT4QA
