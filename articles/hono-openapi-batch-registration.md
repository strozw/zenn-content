---
title: "@hono/zod-openapi の一括ルート登録機能の紹介"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["hono", "openapi", "zod", "typescript"]
published: false
---

ふと `@hono/zod-openapi`  の CHANGELOG を見ていたところ、v1.3.0 で一括ルート登録が可能になっているものの、特に話題になっておらず、もったいなく感じたので、どのような機能で、どのような問題が改善されたのかをまとめました。

## 追加された背景

`@hono/zod-openapi` のバッチ登録機能は以下の PR で、

https://github.com/honojs/middleware/pull/1752

PR には、以下の問題を改善するために追加したと記載されています。

- 複数のルート登録を個別に行うには、繰り返し作業が多く、コードが冗長になっていた
- 複雑なルート構成の型推論が困難だった
- 複数のファイルにまたがるルートの管理が難しかった
- 条件付きのルート登録についてのサポートがなかった
- RPCの型安全性は、分散したルート登録全体で維持するのが困難だった

大きく以下の2つに分類できると思います。

- ルート定義の簡素化と管理の改善
- 型推論の改善


## 追加されたAPI

### `defineOpenAPIRoute`

`defineOpenAPIRoute` は、openapi ルートの定義を簡素化するためのユーティリティ関数で、以前からあった `createRoute` とは異なり、ルートの設定のみでなく、ハンドラーも同時に定義できるようになっています。

```typescript
import { defineOpenAPIRoute, createRoute } from '@hono/zod-openapi'

const getUserRoute = defineOpenAPIRoute({
  route: createRoute({
    method: 'get',
    path: '/users/{id}',
    request: {
      params: z.object({ id: z.string() }),
    },
    responses: {
      200: {
        content: {
          'application/json': {
            schema: z.object({ id: z.string(), name: z.string() }),
          },
        },
      },
    },
  }),
  handler: (c) => {
    const { id } = c.req.valid('param')
    return c.json({ id, name: 'John Doe' }, 200)
  },
})
```

加えて、 `addRoute: boolean` プロパティで条件付きルートとしての登録も可能になっています。

```typescript
import { defineOpenAPIRoute, createRoute } from '@hono/zod-openapi'

const debugRoute = defineOpenAPIRoute({
  route: createRoute({
    /* ... */
  }),
  handler: (c) => {
    /* ... */
  },
  addRoute: process.env.NODE_ENV === 'development', // 開発環境でのみ追加される
})
```


### `OpenAPIHono#openapiRoutes`

こちらは、`HonoOpenAPI` クラスに追加されたメソッドで、`defineOpenAPIRoute` によって定義された複数のルート(定義とハンドラー)を一括で登録するためのものです。

```typescript
import { OpenAPIHono } from '@hono/zod-openapi'

const app = new OpenAPIHono()

const getUserRoute = defineOpenAPIRoute({
  /* ... */
})

const createUserRoute = defineOpenAPIRoute({
  /* ... */
})

const updateUserRoute = defineOpenAPIRoute({
  /* ... */
})

// 'as const' is important for type inference
app.openapiRoutes([getUserRoute, createUserRoute, updateUserRoute] as const)
```

以下のように、 `defineOpenAPIRoute` で定義されたルートを配列でまとめ管理し、スプレッドで渡すこともできます。

```typescript
import { OpenAPIHono, defineOpenAPIRoute } from '@hono/zod-openapi'

export const userRoutes = [
  const getUserRoute = defineOpenAPIRoute({
    /* ... */
  })

  const createUserRoute = defineOpenAPIRoute({
    /* ... */
  })

  const updateUserRoute = defineOpenAPIRoute({
    /* ... */
  })
] as const

export const postRoutes = [
  /* ... */
] as const

const app = new OpenAPIHono()

app.openapiRoutes([...userRoutes, ...postRoutes] as const)
```

## 改善されたことの確認

### ルート定義の簡素化と管理の改善


```typescript
const app = new OpenAPIHono()

const userRoutes = app.openapi(
  createRoute({
    method: 'get',
    path: '/users/{id}',
  }),
  (c) => {
    /* ... */
   }
);
```


### 型推論





### Playground

#### chanin method

https://www.typescriptlang.org/ja/play/?#code/JYWwDg9gTgLgBAbzgeTAUwHYEEAKBJACQgwgBo4BjKNAQxjQCUIBXe8gLzgF84AzKCCDgByAAIALYhAD07CABMAtBHQYaYYMIDcAKFCRYiOOIrc+AocMklpFADbBMMbTp1oAHgfgViAZ3g4NFA0IL4AyhTiaCA0cAC8cOwAdBAARgBWaBQwABQIOnBwwPIAXIlJzMzFOQCUKarqwHkFhXBgQSFl+a2taiBoZcLFwqQtPcAYg+0w4iNj3KM9HiFgdgMiAIwATNu7c4VcNaOHum6e0N5+8ACqvmhQEVEx8YktKRlZud2txWXJ-lAJgBzWr1TCNZo9QrLcBrQbbADM+1ah0WvRC63+MEBGBBdRU4I0kKhMNW62EACkIOIMHAACIQNDIg5HeY0IGYpIYZggVL3UEEtRE75LdwrOFwAAsWzRLLRhzegohwlu92ENVOOh8GH8cAErDQLyotHoTANxP6MwUgw5zjR03Eg2kADcNtJmHcoL5pAhilxhHAaL5KFc0dQAI7MND+LrzdrBUJlQIJ8KRaI0eVh6OQHXR2M9LYABkL+ah2voGBgpahInUq2AFDowGI0nSvmIwi6cF8aZiZVVD17sS4spRo8K8mjVGAYBgzcmIgYaGxjmdhpmho9atHI5au5Org8XhDOvgdZeGDQAHcUKpcIQpLVFQ0ifNjXRGCx6MSepbJKURFtZlWgdJ1gMKahfBzO5fGrKEixLRB5hrcsnDgmtCmEOsHEbOcWzbDt0IwwoeyeGg-neTJsjyOB+l8Xx2U5AFgVqbhWWInpdw4hZkKhSce0BWd50GKI7DsCBwLHXiuJRdjWhyCganiAA+SgkgIjAaLohiOREtAxIktiWhqZ9CSaJAkks-U2DaOhHREF03S3L0fT9AMgxPXURzgBSlLiVSRU8+AkGKMwEgoJIIySZ0aAceQcmEeMQnVXQemoGBmCgWkIo0mjinIRiyiLcg+nJa47GxGhFBiDAA28hCNT3EzCjBIVzLgSzIq-NByFAhznS2d1PW9X15H9QNg21LzyF8lSkJ6KbgqKeQwrUqKYrihKkpAFL5nSzLsvU9tNJC+QCt0uBirgUrBnKyrqpoWqFku4tGpZUy2pozrrJ62yZidZ0ESG+4Rrciague2b-Pm1pFqMUKeHCyK0HDaLYuqRKOh2t7Wn2rK1Ny07zvWK6bpEO7ggep76te1KjJapVhQ6qzut6uyAclYGXNG8aPLh7yoYCt8rnhlbEbWlG0c2zGE12tLlwOgnjrys7Awu0mMVuirKZquryAaumFQZl92u+1m-vs4QXQAVi50GxvcyaRYFxS5sCuHTtWiL1vR+KZeSnGIIV-GcuVom1ZJwsSs18ntaq3XnoNpqPohCyWYNNn-v6gA2O3XId8H+Zm13ofdkXPfF73JY2jHtrl3Hg8OwnluJoqo+umOVTjqm9ZewscaNjrGdN9ObL6q3nQAdjznnHYhl2-KFhby+Wr3kdRmu-brwO9UbpXiBV1uXuj-otfuhOaf7w3mqHk2vtH37x5dAAOGewb553i8XmHCg91fK-XlLWuWN65BwyiHI6B9w6FWPh3U+sdz6PV7knd6xszL3y6hnC2AMACcb8C4f1PJDEuS9YYrwRkaQBm8togJ3njJuYcW4RzbifMq3cL761psnNBn006YLHuzfqGxCz4N5k7IhC83bCyIRXShPtpbbzpmAxWocoFMJgRreBXdEHU04Vfbht90F8J+pnS2jknLDXzmI+eX8pHLxkf-OR1dfY0NlnQveqiTrqPVu3Mm2idZIMTlw1BhjeHM34Y-QRE9tiiLnkXHyJCf4Q1kUjeRwC3FKN3uAhhaj8rMNgX4im8dAmXwHjfVqqdwkmOwUIoGzl7bWPiYLJJf8KGpOcQo2hmT6H7y8XkjRvjO5FJ7kE-RISKlMzNlgp+rpOb1KsXEz+CTv5lwcW0iWG8XH+2xt0jxkC+mqwGaws+ATdF9zKSnSZD9TEAw2LbeZs9C5LOaas3UKSNlAK3l0vaezm79J8cchBpzkHBPpqEypUyBFZ2ibnB579xHTWWXYshayxZOM2Z0jJPzsm9MPvkzRbCdEgrGWCiZI8Ik3KEdPOFBCEXwEkaXaRbzHHtIxekgOuycWeLxUcuBhLgWjIuTwiF1yanRNfjSxpzzEmvKWusqubKvlYvlly-ZPKAV8pOcUs5KDSXDwwdUmZGw8GSsWRI2xjL7HMvlVQrZijsUqLVdAjVhT2ElL0UK8FVyKVipdEWWJTzzVIstSi61aLWWfNcRyh1EC-mHJdUMt1OrQWDzJQa82MydgBsIYil5TK5Xho+dQ7ZoCsmOrjUfAlWqRmlOvpc8lhqol+sGqawNuaZX5tFmvNJSro0qvLYw-5kdAX+O1cSz1abjEZqbQNOpljHk5vpRa0hv9yGFoVZGkt7jVUVvxYMrRwyOHnLrcK71jboV+rmfO+FNjg0ruSSyotdrvn9tjYO+Nw7NVArHYKk9XqG3TovQNe517aW3rzVagt3aOnsp2TGnJBzK37v5T+2tBjJ1VMA2YgasLQNSqDRB0NUGAE9qjXB19CH1WftdUS396H9VTumTOrY1K8NmvbSszt7yN3FvtRR3FzrqOJto2h8ZDHMNMaA1sCVbG21LrvS0td0HFVkdLT07lgmWFftHTWj1f6MOQsiVJk1snF3EM45BrtJGYO9vIw3Hd76kMjsPe6499G76MahdhhEIjW1mYZfe1p67bWYr7fZgduSP1aZowK0TbFThHguHAGAABPdAcAsBgDAC8VL6AIC8EDFlhL5xDBw3sI4SsLwTAAB5MtgGUjkAAROIGAMAwAlGkNIcSjY7CSBjD54sjXGo6CAA


#### openapiroutes

https://www.typescriptlang.org/ja/play/?#code/JYWwDg9gTgLgBAbzgeTAUwHYEEAKBJACQgwgBo4ATNAM2AzVU1zwCUIBXGNcgYyjQCGXNp25wAXnAC+calAgg4AcgACAC2IQA9OIgUAtBHQYBYYEoDcAKFCRYiOGp7TZ8xUo0ktPADbBMMJZWVmgAHnbwPMQAzvA4AlACINEAyjxqaCACcAC8EgB0EABGAFZoPDAAFAhWcHDAFABcBezsDZUAlIXGpsDVtXVwYAlJzTWDgyYgaM1KDUqkAxN0s8MwagtL0osTYUlgPjPKAIwATGcXm3VSHYs31iHh0JEx8ACq0WhQaRlZuRIDQqlcpVcaDBrNcT5WJQOgAc063UwvX6Ezqe3Ah1mZwAzFdBjcdpMkkcoTD4YijMizKi0RiDkclAApCBqDBwAAiEDQ+Outy2AjhpPyGHYICKX0pPRpYN2oX2WLgABZTkS+USboCqSYaUoPl8lB0HlYohhYnB5KJ-nxBMIOFxadN1npZkLAkS1mpZloAG7HLTsT5QaJaBANKT4-gAR3YaFiYy2w0SyWa8WTqXSmQEGqJ-GikDNcYTE1OAAZS8W0aauBgYJW0cpTAdgDwhMBiFoStFiEoxnBopmss19d9B9kpGqCZO6lQB7CwDB2xhZiw0DBYWgfWg4Ott4GDZOJwMj-dgmEInBTeam-96AB3FDGZhEEidLXS4AiLjRSoAbS2VC0PQjDYPgX5oLSEyWlwzQ2kIaDgZBaJOhoTTKG6vJop63qYVBcYFp80T1g2cBlhWiBbCRl7EDWdYUVRDZKE2fitouHZdj2xEMYMA6-AIkJAmUFTVHA0zRNEgrCuSGAIh00j8txExHop2yUSRs58MAC5LrMGQ+D4EC4WiykkSZSkKWiagCBgFCHFAzSVDwck5AAfJe+QcRgIliRJQq6Wg+mGfJWyEgBNB0AwT5gfaEGyoM0FHEg+TJQl5DYcovr+vuwahuGShwAI0TUWa8ATnAalWTZdkOU5uRuXFExXvASANC4eQ8Pk0b5D6Ah+BQlRKEmSSGtYVH8DA7BQOyHWeSJDTkJJzRluQUyMm8PjrgI+hZBg+VlWRRpqVIIUWTO4XAVFrAxUhdQJX2yWdTFaVCF6GU+qcAZBiGYYUBGBVFU12zlQ2lW2V8NXOfVal1IDLUUG17ldT1fUDUNIAjdDFprpN00ed2XlwwtfmkaWK0krM62bdt1l7eQB2jcZJ2ToBEUgcwiENRaMX3SlT1DC93o+jin1fN9eX-cV5plRV1lg-ZcCOZD9EkbD9TwzI7WdWgUbdb17SDSM6OHWN2NTe5s2EwVxPLXAq0UxtiTU7tQP00dTNhUBkVMNFog3Vzog849ojPesgtKiLOU-X9hWS6V5Ay1V4MK7VrnKw2qutRriPa7rKMG8mGMmxNZszfjc0UETRw23byiU47O20yTpbG4z5nM+dXugVdvuc3dDgPal-Oh29ACsEdi79+Ux4D0sg7L1XJ0rnMw68DiZ9aWs68j+to4XJHjTj5tl5bi0k2T0z21TDcu+WLdKe7Ewsxd3vdw6vfc-3vPB0Pr1KL6ABs49cqTwljPeOc9E7y0VnVNOVZV5wwRh1JGet+r52GnfNEB8S542IOXSuS1Sa23JrXB2W1r77Vvgze+bcPas0uhzNSfckpfy4CHX+voADsQCo5TwBqvWell55J2ganZesc17qw3sgvOu8MF4WLrjC2at8FnyIRfEhV8aY32blQgkD9BhP07uza678A6fyDqwn+gsAAc3DxbT34eAwRkCIYwLERnSRmtpE70NnvBsWDFHH2UVbKuhCa56lIU7RursGzHRoY-DubMfZv0YR-ZhFixDpT-j6AAnHYkBDiSpAwTnLVxojMYeMQZvXOPiC5yPiqbQJuCT7WzCcQiJmjnYUJ0W7eJBjEn0JMaksx6TB5ZMyqWfJ0c+FFIERMUGC8RFQyopUrOSCc7b1QbI3R8jD6l2acE0+1d2l1zIVo7p9S4kElOpQAZL8GH+LSXAAefNxl+iyl9YB0zxFzMGAs4RKdlkq3gWrKp3itm+PqbdRpR8DnzRCQQ8+a1InkLppQ3p1z26eySa-WKwyYLmLGQLN6Zwpm8J+U4+ZQioGAtgY1EF68vEbJQajSFOyGkKNhQTQ5rSkWX3ructFPTYn6LOtiwZPd8WJWeSwzJxLsnHGFtlCe3ywHA2caUxebiKkMs8dnLeLK0FG3ZdCzl+zuXwqOW09RHSBVdKFZc0VtzxX3KGY8kZMqMlsMFsccOyqvnkrVSUxZtL3G6rBcymRbLMYBK5XghFqjwmnKidox1fSxV0NdZK91BLRmvPlZlMe-qeGgMceqqlLitXlJWeGtZ1TNmsrqSarGZqcEWorgm45Nrk2oqbmmzFtDn5dweZgp5Lzv5vOOIA4t9iZlS0pX86lZSgXp1rVIyNtT0HNtjea+NVq+UaLtdE9FIr03OszcOt1o6PXjssZOrhM6ClzrjuWxdlall0sGKs9dBqo1NpjTC3dLTQkHttWc+1fb2VXPVIOoxyS8U5ulbeuVw8FW2MfaqstwaAVLx1UUhBdbwWNq3QB1tSjLW8rUcizpx7hWtwHQkl1l7s3XtzZ6olqHMp5Iw4GrDEDNUfrDfh0FhGN0Qv-UXPZba92UaTSiwVkGMUwcYxe4xLGoJjtld6t6ZYyWltmQuuo-yaW4ZrcJxl+qaniZI5J7B5GO37qo-y8DtH+3Kf6UxtTKTEOBw4+w96HzRYBv0-O19Rml1VpXXA8zer1m-s3ca0jUn7MqK7dRo9qaoNOsMTikdGmb1aasTpj6PGQsvuwyZ7VZnzQEZ-VZ4jiXbNNPbal616WXOZaU-JLFqn4N+yYex-NnH3pKs+SWwpoWKvLs-SvGLEb4vWca-vQD0ngOIqc4ejrFystnpyxK7zrGkOFbeacP1Y3Z0UrC44CLgm8M1ZE3VhtRq-GYJWylztbXnMpu2110KKmh1eYQ4d3zQ3-OnCLedp9l2puRZm+I2rTKFsNZe7suzQSKMgY22B77DqdsMY8713F-XNNeqK9k0407IeYYM1d4z02hP3Ys3F+rz2oUtuS+jhzsmTnyYgzE+j7mM0A766YtjyHtPk4fVT3jNOYe3eq81B7iOWfbKS2juFXPMdyZo5109+OhdwaJ6Lo7pOTvoel2V4p-GQ2meBXN0TSPWfbre5z1roGe0Kf59Q-X57hdG6lSDidBb3rcYtxN8r1ucNVbt4z2L9bDWq6a3Gtbiaec65+3rwXvvDd5fiiTvzgscSTNK+Hq3GqbfR9Xfbx7Cfo1J6AzyrXaeMsZ4F91gYABdMrh1HgXhgAAT3QHALAYAwD-AH+gCA1ACqj4eOeZ44jfD+FrP8JwAAeEfYAXKVAAERqBgDAMAjQtBaAMq2HwGh4xF-LDv42VggA
