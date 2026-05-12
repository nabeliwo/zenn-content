---
title: "props の改善から始めて漸進的に考える React コンポーネント設計論"
emoji: "⚛️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "react"]
published: false
---

突然ですが、こんな React コンポーネントの props を見たとき、あなたはどう思いますか？

```tsx
type Props = {
  data?: User[]
  isLoading?: boolean
  error?: Error
}
```

「親コンポーネントで API を叩いているんやろなあ」とか「`useState` で loading や error を管理しているんやろなあ」くらいはなんとなく想像できそうですね。

実際いろいろなところでこういうコードはよく見ます。

では、この props を受け取るコンポーネントにおいて、次の状態はあり得るでしょうか？

```tsx
<UserList
  data={users}
  isLoading
  error={error}
/>
```

型定義としては許されますね。ただこれは一体どんな状態なんだよって感じがします。
読み込み中なのに data があり、同時に error もあります。実際の UI としてこの状態を表現したいケースは思いつかなそうです。それでも、この型定義では書けてしまいますね。

`UserList` コンポーネント側の実装も、こんな感じになるんだろうなあと想像できます。

```tsx
const UserList = ({ data, isLoading, error }: Props) => {
  if (isLoading) return <Loader />

  if (error) return <ErrorMessage error={error} />

  return data?.map((item) => (
    <div key={item.id}>
      {/* 省略 */}
    </div>
  ))
}
```

しかし、この props からは `UserList` が取りうる UI の状態があまり見えてきません。
data、isLoading、error という値は並んでいますが、それらがどのような関係にあり、どの組み合わせが正しいのかは props の型から読み取れません。
つまり、この props は UI の状態を表しているというより、ただの値の集合になっています。

コンポーネントが取りうる状態や振る舞いを、props からもっと直感的に読み取れると良さそうです。
…良さそうじゃないですか？良さそうですよね。

この記事では、先に挙げた例の改善から始めて、React コンポーネントの props 設計をどのように考えると良いのかを掘り下げていきます。

## 「UI の状態を表す」とは

UI の状態を表せてないなどとのたまっておりますが、僕がそれをどんな意味を込めて言っているのかという話をします。
先の `UserList` を例にして考えた場合、このコンポーネントが本当に受け取りたいのは、`data`、`isLoading`、`error` という3つの独立した値ではなく、次のような3つの状態のいずれかです。

```tsx
<UserList data={users} />   // データ取得に成功した
<UserList isLoading />      // データを取得中
<UserList error={error} />  // データ取得に失敗した
```

ただ現在の型定義だと、それらが独立した optional な props として定義されているため、実際にはありえない状態が簡単に表現できてしまいます。

## discriminated union で意図を明確にする

そこで、props を独立した値の集合として扱うのではなく、コンポーネントが取りうる状態として宣言し直してみます。

`status` を判別用のプロパティとして持たせ、それぞれの状態で必要な値だけを定義します。

```tsx
type Props =
  | { status: "loading" }
  | { status: "error"; error: Error }
  | { status: "success"; data: User[] }
```

この型定義の場合、`loading` 状態では `data` や `error` を渡せません。
同じように、`success` 状態では `data` が必須になり、`error` は渡せません。

つまり、実際の UI として存在してほしくない不正な状態を、props の型レベルで防ぐことができます。
結果として以下の3パターンの状態に絞ることができます。

```tsx
<UserList status="loading" />
<UserList status="error" error={error} />
<UserList status="success" data={users} />
```

そしてこの props はコンポーネント側の実装にも利点があります。

```tsx
const UserList = (props: Props) => {
  switch (props.status) {
    case "loading":
      return <Loader />

    case "error":
      return <ErrorMessage error={props.error} />

    case "success": {
      return props.data.map((item) => (
        <div key={item.id}>
          {/* 省略 */}
        </div>
      ))
    }
  }
}
```

switch 文か if 文かというのはどちらでも良いのですが、ポイントは、最初のコード例にあった `data?.map(...)` のような存在するかわからない値に対する防御的なコードが減ることです。

このように discriminated union を使うことで、`UserList` が取りうる状態や振る舞いを props の型として表現できるようになります。

結果として、コンポーネント使用者はどの状態を渡せば良いかがわかりやすくなり、コンポーネント実装者は存在しない状態を考慮しなくてよくなる。というところで、Win-Win です。みんなハッピーですね。

言い換えると、props は単なる値の集合ではなく、UI の状態を宣言するためのものとして捉え直すことができます。
そしてこの捉え方は、コンポーネントが何を担うのか、つまりコンポーネントの責務を props でどう表現するか、という設計の話にもつながっていきます。

## 日常に潜むあり得ない状態の発見と改善

他にも、props が UI の状態を表現できていないケースはよく見られます。いくつか例を見てみます。

### `Button` の見た目を変える props

例えば、`Button` コンポーネントの見た目のバリエーションを `primary` / `secondary` / `danger` という props で変更できるようにしたいとき。

```tsx
type Props = {
  primary?: boolean
  secondary?: boolean
  danger?: boolean
}
```

一見すると問題なさそうですが、この型定義だと次のような状態も表現できてしまいます。

```tsx
<Button
  primary
  secondary
  danger
/>
```

これは一体どの見た目になるのでしょうか？

実際には、`Button` の見た目は `primary` / `secondary` / `danger` のどれか1つであってほしいはずです。
つまり、この props も複数の独立した値ではなく、見た目の状態を表現したいケースです。

その場合は、次のように variant を使って状態を1つにまとめる方が、コンポーネントの意図が明確になります。

```tsx
type Props = {
  variant: "primary" | "secondary" | "danger"
}
```

これなら、`Button` が取りうる状態を型から直感的に読み取れるようになります。

### `LinkButton` の挙動を変える props

次は `LinkButton` のようなコンポーネントです。
ここでは、内部実装としてリンク (`<a>`) かボタン (`<button>`) のどちらに描画されるかを props で切り替える、単一のコンポーネントを想定しています。
(そもそもこのようなコンポーネントが適切かどうかという話があるのですが、それは置いておきましょう。)

```tsx
type Props = {
  href?: string
  onClick?: () => void
}
```

これも一見自然ですが、次のような状態を許してしまいます。

```tsx
<LinkButton href="https://example.com" onClick={() => handleClick()} />
```

これはリンクなのでしょうか？ボタンなのでしょうか？

もちろん、リンクとして遷移しつつクリック時に何か処理をしたいというケースもあり得るので、決して間違っているわけではないです。
ただ、コンポーネント設計としては、リンクとして振る舞うのかボタンとして振る舞うのかを、props からもっと明確に読み取れる方が嬉しいことが多いです。

その場合は、次のように排他的な props として定義できます。

```tsx
type Props =
  | { href: string; onClick?: never }
  | { onClick: () => void; href?: never }
```

never を含むユニオン型を定義することで、`href` を持つならリンクとして振る舞う、`onClick` を持つならボタンとして振る舞う、という意図を型で表現できます。

### controlled なのか uncontrolled なのか

先ほどの `LinkButton` と同じ `never` で排他的にするパターンが、フォームコンポーネントにも応用できます。

```tsx
type Props = {
  value?: string
  defaultValue?: string
}
```

React に慣れている人なら、これを見て controlled / uncontrolled component の話だとわかるかもしれません。
しかしこの props も、

```tsx
<TextInput value="hello" defaultValue="world" />
```

のような状態を許してしまいます。
(input に対してこのような props を渡した場合は、ログに warning が出力されますね。)

実際には、`value` を使う controlled component と `defaultValue` を使う uncontrolled component は異なる状態・責務を持っています。

その場合は、次のように状態を分けて表現できます。

```tsx
type Props =
  | {
      value: string
      onChange: (value: string) => void
      defaultValue?: never
    }
  | {
      defaultValue?: string
      value?: never
      onChange?: never
    }
```

これによって、

- controlled と uncontrolled を同時に使えない
- どちらの使い方をしたいコンポーネントなのかがわかりやすい

という利点があります。

## コンポーネント分割で構造そのものを表現する

ここまで、discriminated union や `never` を使って、props の型定義によってコンポーネントの状態や責務を表現する方法を見てきました。
ただ、React コンポーネントの設計では、常に型で頑張るのが正解というわけでもありません。
場合によっては、そもそも props として渡さないという方向で設計した方が、UI の構造を自然に表現できることがあります。

例えば `Card` コンポーネントを考えてみます。

```tsx
<Card
  title="Profile"
  subtitle="Settings"
  actions={<Menu />}
  footer={<Button>Save</Button>}
>
  {/* 省略 */}
</Card>
```

これも決して悪い設計というわけではありません。
ただ、`Card` の中身が複雑になっていくと `title`, `subtitle`, `actions`, `footer` のような props が増えていき、`Card` がどのような構造を持つコンポーネントなのかが props の一覧からは少し見えづらくなっていきます。

`Card` コンポーネントの利用箇所が増えるたびに、些細な違いを表現するために boolean props が増えていき、内部実装が複雑化する未来も見えてきます。
…見えてきますよね？僕には見えてきました。

こういったケースでは、props の型定義をさらに工夫するよりも、UI の構造そのものを JSX で表現した方が自然なことがあります。
例えば [Compound Components](https://zenn.dev/tsuboi/articles/d922a3e8c5f53e) を使うと、次のように書けます。

```tsx
<Card>
  <Card.Header>
    <Card.Title>Profile</Card.Title>
    <Card.Subtitle>Settings</Card.Subtitle>

    <Card.Actions>
      <Menu />
    </Card.Actions>
  </Card.Header>

  <Card.Body>
    {/* 省略 */}
  </Card.Body>

  <Card.Footer>
    <Button>Save</Button>
  </Card.Footer>
</Card>
```

この形にすると、`title` や `footer` のような値を props として渡すのではなく、`Card` の構造そのものを JSX として表現できます。

- Header がある
- Title がある
- Actions がある
- Body がある
- Footer がある

ということが、コンポーネントの利用側からも読み取りやすくなります。
利用側の JSX がそのまま `Card` の構造を表すドキュメントになるとも言えますね。
また、各パーツを使う/使わないも JSX として書くか書かないかで自然に表現できますし、配置や組み合わせを変えたいときも `Card` 本体に boolean props を増やすのではなく、JSX 側で組み替えて表現できます。
結果として、`Card` の利用箇所が増えても、コンポーネント本体は最小限の責務に留めやすくなります。

ここで大事なのは、すべてを Compound Components にすれば良いという話ではないです。
単純な値や設定として渡した方が自然なものもありますし、データの配列を受け取って一覧を描画するようなケースでは、props として渡した方が扱いやすいことも多いです。

ただ、props が増えてきたときに、props の型の制御で状態を表現できるのか、それとも JSX の構造として表現した方が良いのか、みたいなことを考えてみると、設計の選択肢が増えます。
どちらも、コンポーネントが取りうる状態や責務をよりわかりやすくするための手段だと思います。

## まとめ

この記事では、props の改善を入口にして、React コンポーネントの設計について考えてみました。
最初に見た `UserList` の例では `data`, `isLoading`, `error` がそれぞれ独立した値として定義されていました。

しかし実際には、それらは単なる値ではなく、データを取得中 / データ取得に失敗した / データ取得に成功した、といったコンポーネントの状態を表現するためのものでした。

そのように考えてみると、props 設計は単にどんな値を受け取るかを決めるものではなく、コンポーネントがどんな状態を取りうるのか、どの組み合わせが存在してよいのか、どんな責務を持っているのか、ということを表現するための設計でもあることが見えてきます。
そのための手段として、状況に応じて次のような選択肢があります。

- 状態が排他的なとき: discriminated union で状態のパターンを宣言する
- 同時に持てない props があるとき: `never` で組み合わせを制約する
- props が増えて構造が見えづらいとき: Compound Components で UI の構造を JSX として表現する

もちろん、常に1つの正解があるわけではありません。
ただ、props を単なる値の集合として扱うのではなく、「この props はコンポーネントの状態や責務を正しく表現できているだろうか？」という視点で見てみると、React コンポーネント設計についてまた違った見え方ができるかもしれません。
