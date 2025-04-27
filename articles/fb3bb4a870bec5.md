---
title: "unionなオブジェクト型をPickするときはDistributive Conditional TypeでPickする"
emoji: "🤏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

## 始めに

Reactでpropsを一部pickした際にunionで定義した情報が消えてしまい条件分岐によって型が変わらなくなってしまっていました。

```tsx:pickをそのまま使うとunionの情報がなくなる
import { pick } from 'lodash-es'

type Props = {
  label: string
  required?: boolean
} & (
  | {
      multiple?: false
      value: number | null
      onChangeValue: (newValue: number | null) => void
    }
  | {
      multiple: true
      value: number[]
      onChangeValue: (newValue: number[]) => void
    }
)

const Component: FC<Props> = (props) => {
  const conditionalProps = pick(props, ['multiple', 'value', 'onChangeValue'])

  const handleClear = () => {
    // 条件分岐によってonChangeValueの型が変わらなくなってエラーになる
    if (conditionalProps.multiple) {
      conditionalProps.onChangeValue([])
    } else {
      conditionalProps.onChangeValue(null)
    }
  }
}
```

理由は単純で`pick`メソッドの返り値である`Pick`は以下のような実装になっているのでオブジェクト単位でのunionが考慮できずキーごとのunionに変換されてしまいます。

```ts
// Pickの内部実装イメージ
type Pick<T, K extends keyof T> = {
  [Key in K]: T[Key]
}

// 従って Pick<Props, 'multiple' | 'value' | 'onChangeValue'> の結果は
// 以下のようにキーごとのunionになる
type ConditionalProps = {
  multiple: Props['multiple'] // (false | undefined) | true
  value: Props['value'] // (number | null) | number[]
  onChangeValue: Props['onChangeValue'] // ((newValue: number | null) => void) | ((newValue: number[]) => void) 
}
```

pickせずpropsのまま扱ったり、conditional部分以外のpropだけ分割代入して、conditionalな部分だけrestPropsとして書く運用でもなんとかなりますが、個人的にpickでconditionalな状態を維持することができないか気になったので調査して記事にまとめました。

## Distributive Conditional Typeを使ってunion状態を維持してPickする

結論から言うと以下のようにDistributive Conditional Typeを使ってそれぞれにPickが当たるようにします。

```ts:unionなオブジェクトに対して、それぞれPickする
type DistributivePick<
  T extends object,
  K extends keyof T
> = T extends any ? Pick<T, K> : never

// 例のPropsに使用すると以下のようにそれぞれにPickが分配される
type ConditionalProps =
  | Pick<{
      label: string
      required?: boolean
    } & {
      multiple?: false
      value: number | null
      onChangeValue: (newValue: number | null) => void
    }, 'multiple' | 'value' | 'onChangeValue'>
  | Pick<{
      label: string
      required?: boolean
    } & {
      multiple: true
      value: number[]
      onChangeValue: (newValue: number[]) => void
    }, 'multiple' | 'value' | 'onChangeValue'>
```

Distributive Conditional Typeについては以下の記事とかを参考にすると良いと思います。

https://zenn.dev/hrkmtsmt/articles/be9a20fa7d3aaf

また余談ですが、この方法を`Omit`のケースでMUIでもやっているようで、以下にユーティリティとして用意されていました。

https://github.com/mui/material-ui/blob/v6.4.11/packages/mui-types/index.d.ts#L37-L43

## 終わりに

以上がunionなオブジェクト型をpickする時の方法でした。検証コードをTypeScript Playgroundで書いたので、推論の様子とか気になる方はこちらもご参照いただければと思います。

https://www.typescriptlang.org/play/?#code/JYWwDg9gTgLgBAbzmYBjA1nAvnAZlCEOAcgBsIATAQwGcALAWgFMbiAoNgegCpu4BXAHbAIgwFYMgKoZAawyAOhkDlDIHqGQBMMgawZAf9qB1BkBmDIEAGQLoMgGIZAegwGACmnSBNBkDRDHG6c2MAJ5gmcACLAaMKMABG-DDAAG5M5hgAPGxwcAAqcEwAHjBMghQ0cBB+AFZMqDAANNFwANIJyanpcOhMThC4cWwAfHAAvHHlKWkZVIJOcAD8cOHoEbEFpS0AXHCCTKFQHDzc0XxCIuLS8sqA0eqASQyAwAGAdoYagPoMKBirDqii3nAUXj7+gSFhFgDyOXnw7VEx8UkulUsrl8kUYmVAZUMjU6g1Ys0ABQgmbjaq1GgzEoAbQAugBKGaeby+AJBUIjMYTEotVotBDFG6CO4XGoUT7ZNrICzInITWE0fHFKBMGD8KCCbkYJjsnJwWgeR6kl4UixUyZsLAcZyuYYEMAZdoMmKkKh+JikGYk4CCADmwqYAEd+MARRQBjM-BAIKQmL1NXAAGRwRHFAA+iGKMRiIH4pCCYF9HrwVFINCYUejwVT-CYM0E-BA5qgcAjBdIpEzMVEAGE6L1bUwAGo5vMhuYAdxbpFz+cLxdLszjpHxbRawQgwAoma1MQjxujMbjCd9Mx8uarcGzPbbBaLTCgeM3tfrdubrZmiM73d7Q-3h4JY63k+ni61QrYTJZ+sxeogBq5BdTXNS0SEAboZABuGZRAB2GQBhhgUQBOhmIcE4FjeNgETNt1yYFDt1vPEUJPBtzx3S9r1bQUnwXGIvx9JgADpyFtK8mC7CihRiLUcAVUwfzgThODiFwmAAZVQXwwBgQA7BkACuNADWowAZBkAKkVAAHPQBzBj0KTAD8GLTAGO5QBTRRsQALBjQlcsKgXM1MAL7VACztQBVBgMLQNC0wBjBlOKS1NoQBahkAY4YEKUNT1lEPZAFrfQBAYxsDgkAEqV0D2TSdMAf3lADEGRlbngVkZRrUQHiCURU14-9DVixEwB-CZsWIUyMN9ZCSDwpg6uIIizxvRqCWKTKKGytJgDywQCp-eiWsbNrihioLBCkwBRg0ARg0FMACNtAAkGJzXK0QBDc0AN7lADAlQAEnRUQAIhmsHZ5IUtTAFkGMRABEGYpgAaUqLCynK+o2Qaivo6rMNHajYqe3r+reg1hsEOtiLaxE8Q47AEjTNwfq6nrcte0hCqBkaSNzK9hyhriov4wTiSeMlXhGPZEpcrRWUimj0t+7rnoBlG+PaB5rWJ1UMA5b5SvKuBKs+2qJmIBqmvRtriA6mIJuEUQZvm8KHK0K7rFOQBAhh0VltEAIIYtr2w7jtOi7rFu+6EYZ5HUZoD7lxqphvszM3-otoaxdbCHJehi100jRdHaR-Kmfe12dyxiscc1NggA
