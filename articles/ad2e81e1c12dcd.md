---
title: "React Hook Formで考える型安全なフォーム設計"
emoji: "🤔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React","TypeScript","ReactHookForm"]
published: false
---

こんにちは、かるカンです。
今回は、React Hook Formを使って実装するフォームの設計案について話していきます。

# これはなに ?
- React Hook Formを使って実装するフォームの設計案
- React Hook Formをこれから採用するけど、フォーム設計どうするか検討している方の参考になれば
- React Hook Formの基本的な使い方については触れないので、キャッチアップ後に読むことをオススメします

# ライブラリのバージョン
- React: 18.2.0
- React Hook Form: 7.39.5
- Zod: 3.19.1
- @hookform/resolvers: 2.9.10

# この記事のゴール
- React Hook Formで型安全なフォームを実装する
- フォームの定義、バリデーションを一箇所に集約する
- ライブラリへの依存を特定のファイルに閉じ込める

# 背景
私が携わっているプロダクトにおいて、新たにNext.jsを採用することになり、フォームの実装にReact Hook Formを採用することになりました。しかし、私自身React Hook Formは個人で利用したことはあったものの、設計して運用しているわけではありませんでした。
そのため、型安全性を保つにはどうすればいいか、開発体験を良くするにはどうすればいいか考える必要がありました。
技術記事としてまとめるのは初なので、生暖かい目で見守っていただければと思います。

# 何が問題なのか？
React Hook Formでは、`useForm`というHookを使用してフォームに必要なオブジェクトを取得することができます。

```tsx
import { FC } from 'react'
import { SubmitHandler, useForm } from 'react-hook-form'

type SchemaType = {
  email: string
  password: string
}

export const Form: FC = () => {
  const submitHandler: SubmitHandler<SchemaType> = (values) => {
    alert(`email: ${values.email}\npassword: ${values.password}`)
  }

  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<SchemaType>({
    defaultValues: {
      email: '',
      password: '',
    },
  })

  return (
    <form onSubmit={handleSubmit(submitHandler)}>
      <label>
        <input type={'email'} {...register('email')} />
      </label>
      {errors?.email && <p>{errors.email.message}</p>}
      <label>
        <input type={'password'} {...register('password')} />
      </label>
      {errors?.password && <p>{errors.password.message}</p>}
    </form>
  )
}
```
また、`resolver`を利用することでフォームのバリデーションに他のバリデーションライブラリを組み込むことができます。今回はZodを組み込みます。
```tsx
import { FC } from 'react'
import { zodResolver } from '@hookform/resolvers/zod'
import { SubmitHandler, useForm } from 'react-hook-form'
import * as z from 'zod'

type SchemaType = {
  email: string
  password: string
}

const validationSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1).max(8),
})

export const Form: FC = () => {
  const submitHandler: SubmitHandler<SchemaType> = (values) => {
    alert(`email: ${values.email}\npassword: ${values.password}`)
  }

  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<SchemaType>({
    defaultValues: {
      email: '',
      password: '',
    },
    resolver: zodResolver(validationSchema),
  })

  return (
    <form onSubmit={handleSubmit(submitHandler)}>
      <label>
        <input type={'email'} {...register('email')} />
      </label>
      {errors?.email && <p>{errors.email.message}</p>}
      <label>
        <input type={'password'} {...register('password')} />
      </label>
      {errors?.password && <p>{errors.password.message}</p>}
    </form>
  )
}
```
一見、このまま実装しても問題なさそうに見えますが、いくつか課題があります。初見だと私も問題なさそうに感じてましたが、以下の記事を読んでみていくつか課題があることがわかりました。(大変参考になるので、こちらの記事も読むことをオススメします。)
https://zenn.dev/yuitosato/articles/292f13816993ef
https://hireroo.io/journal/tech/react-hook-form-within-mono-repo
## 課題
- `useForm`が型安全でない
- フォームにバリデーションスキーマが依存している
- フォームにZodが依存している

# useFormを型安全にする
`useForm`の`defaultValues`の型定義がゆるく、型通りの初期化を強制できません。以下の例だと、passwordを初期化していませんが、コンパイルを通ってしまいます。実行前の静的解析の時点でここを潰しておきたいです。
```tsx
type SchemaType = {
  email: string
  password: string
}
・・・
const {
  register,
  handleSubmit,
  formState: { errors },
} = useForm<SchemaType>({
  defaultValues: {
    email: '',
    // passwordを初期化していないが、エラーにならない
  },
  resolver: zodResolver(validationSchema),
})
```

この観点については、こちらの記事を参考にさせていただきました。
https://zenn.dev/yuitosato/articles/292f13816993ef#1.-useform%E3%82%92%E3%83%A9%E3%83%83%E3%83%97%E3%81%97%E3%81%A6%E3%82%BF%E3%82%A4%E3%83%97%E3%82%BB%E3%83%BC%E3%83%95%E3%81%AB%E3%81%99%E3%82%8B

型安全にするために、`useForm`をラップして新たに定義します。`defaultValues`を`Partial<FORM_TYPE> | undefined`ではなく`FORM_TYPE`に上書きすることで、初期化を強制しています。
また、`UseFormProps`のジェネリクスの型は`FieldValues`で型定義は`[x: string]: any`となっています。any型を含んでおり、想定外の使用をされる危険性があるので可能であれば潰しておきたいです。ということで、`Record<string, unknown>`と定義してany型を潰しています。
```tsx
import { useForm, UseFormProps, UseFormReturn } from 'react-hook-form'

export const useDefaultForm = <
  FORM_TYPE extends Record<string, unknown>,
>(
  options: UseFormProps<FORM_TYPE> & {
    defaultValues: FORM_TYPE
  },
): UseFormReturn<FORM_TYPE> => {
  return useForm<FORM_TYPE>({
    // フォーム全体に設定したいオプションを追加
    mode: 'onChange',
    // フォームごとで設定したオプションを追加
    ...options,
  })
}
```

# フォームからバリデーションスキーマを切り離す
この観点での定義については、こちらの記事を参考にさせていただきました。
https://hireroo.io/journal/tech/react-hook-form-within-mono-repo
フォームにバリデーションスキーマが依存していることの問題点として、同じスキーマのフォームでもUIが異なれば別途定義する必要があり、同じバリデーションスキーマが複数定義されることにあります。
そこでバリデーションスキーマを別ファイルに切り出し、importして利用する形式にします。
```ts
import * as z from 'zod'

export const useSampletFormSchema = () => {
  return z.object({
    email: z.string().email(),
    password: z.string().min(1).max(8),
  })
}

export type SampleFormValidationSchema = ReturnType<typeof useSampletFormSchema>
export type SampleFormSchema = z.infer<SampleFormValidationSchema>
```
`z.infer`でバリデーションスキーマからフォームの型を生成することで、バリデーションスキーマとフォームのプロパティが必ず一致し、ミスを防止できます。
:::message
`z.infer`は出力時の型を生成するため、`transform`と組み合わせて使用するときは注意が必要です。
:::
```ts
export type SampleFormValidationSchema = ReturnType<typeof useSampletFormSchema>
// バリデーションスキーマからフォームの型を定義
export type SampleFormSchema = z.infer<SampleFormValidationSchema>
```
切り離したバリデーションスキーマをラップした`useForm`に組み込んでいきます。
`resolver`を追加することで、使う側はバリデーションスキーマを渡すだけで良くなり、`zodResolver`を使うたびに呼び出す必要がなくなります。
Omit型で`UseFormProps`から`resolver`型を取り除くことで、外部から`resolver`を設定できないように型で守り、意図しない利用を防いでいます。
```ts
import { zodResolver } from '@hookform/resolvers/zod'
import { useForm, UseFormProps, UseFormReturn } from 'react-hook-form'
import { ZodType, ZodTypeDef } from 'zod'

export const useDefaultForm = <
  FORM_TYPE extends Record<string, unknown>,
  VALIDATION_SCHEMA extends ZodType<unknown, ZodTypeDef, unknown> = ZodType<
    unknown,
    ZodTypeDef,
    unknown
  >,
>(
  options: Omit<UseFormProps<FORM_TYPE>, 'resolver'> & {
    defaultValues: FORM_TYPE
  },
  validationSchema: VALIDATION_SCHEMA,
): UseFormReturn<FORM_TYPE> => {
  return useForm<FORM_TYPE>({
    mode: 'onChange',
    ...options,
    resolver: zodResolver(validationSchema),
  })
}
```
呼び出す側のコードは、`useDefaultForm`とバリデーションスキーマを呼び出し利用するだけで済みます。また、Zodに関する記述は`useDefaultForm`とバリデーションスキーマに集約されているため、利用する側はZodに依存していません。
```tsx
import { FC } from 'react'
import { SubmitHandler } from 'react-hook-form'
import { useDefaultForm } from '@/libs/react-hook-form'
import {
  useSampletFormSchema,
  SampleFormSchema,
  SampleFormValidationSchema,
} from '../schema/SampleFormSchema'

export const Form: FC = () => {
  const submitHandler: SubmitHandler<SampleFormSchema> = (values) => {
    alert(`email: ${values.email}\npassword: ${values.password}`)
  }

  // Zodのバリデーションスキーマを生成
  const validationSchema = useSampletFormSchema()

  const {
    register,
    formState: { errors },
    handleSubmit,
  } = useDefaultForm<SampleFormSchema, SampleFormValidationSchema>(
    {
      defaultValues: {
        email: '',
        password: '',
      },
    },
    // Zodのバリデーションスキーマを渡す
    validationSchema,
  )

  return (
    <form onSubmit={handleSubmit(submitHandler)}>
      <label>
        <input type={'email'} {...register('email')} />
      </label>
      {errors?.email && <p>{errors.email.message}</p>}
      <label>
        <input type={'password'} {...register('password')} />
      </label>
      {errors?.password && <p>{errors.password.message}</p>}
    </form>
  )
}
```
# おわりに
React Hook Formで考える型安全なフォーム設計、いかがでしたでしょうか？
意見や感想をコメントいただけると喜びます。
これを見た人の参考に少しでもなれば。

# 参考
https://hireroo.io/journal/tech/react-hook-form-within-mono-repo
https://zenn.dev/yuitosato/articles/292f13816993ef
https://react-hook-form.com/api/useform/
https://github.com/colinhacks/zod
https://github.com/alan2207/bulletproof-react
