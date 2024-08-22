---
title: "AstroでPHPを出力する"
emoji: "🕊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["astro", "php", "wordpress"]
published: true
---

AstroでWordPressテーマを作成する際に使用した小物類の備忘録。

## PHPタグ出力コンポーネント

slotの中身をエスケープせずに `<?php` と `?>` の間に展開するだけの単純なもの。

```tsx:PHP.astro
---
const php = await Astro.slots.render('default');
---

<Fragment set:html={`<?php ${php} ?>`} />
```

- 参照: [set:html (Astro Docs)](https://docs.astro.build/ja/reference/directives-reference/#sethtml)
- `replace(/\s+/gm, ' ')` などに通して簡易的にminifyするのもアリ

### 使用方法

PHPタグの代わりに使用する。

```tsx:index.astro
---
import PHP from 'path/to/PHP.astro';
---

<PHP>if($foo):</PHP>
  <p><PHP>echo $foo;</PHP></p>
<PHP>endif;</PHP>
```

出力は以下のようになる。

```php
<?php if($foo): ?>
  <p><?php echo $foo; ?></p>
<?php endif; ?>
```

エスケープが必要な文字を含む場合は `is:raw` ディレクティブを使用する。

```tsx:index.astro
<PHP is:raw>foreach($items as $i => $item):</PHP>
  <PHP>if($i !== 0):</PHP>
    <p><PHP>echo $item['foo'];</PHP></p>
    <p><PHP>echo $item['bar'];</PHP></p>
  <PHP>endif;</PHP>
<PHP>endforeach;</PHP>
```

- 参照: [is:raw (Astro Docs)](https://docs.astro.build/ja/reference/directives-reference/#israw)

## コンポーネント間のPHPタグの受け渡し

細かく検証していないが、この程度の内容であればそのままで問題ない。

```tsx:index.astro
<PHP>
  // ...
  $title = $foo . $bar;
</PHP>
<Layout
  title='<?php echo $title; ?>'
  permalink='<?php the_permalink(); ?>'
>
...
</Layout>
```

```tsx:Layout.astro
---
interface Props {
  title: string;
  parmalink: string;
}

const { title, permalink } = Astro.props;
---

// （中略）

<title set:html={title} />
<meta property='og:url' content={permalink} />
```

title要素については `title='echo $title;'` を渡して `<title><PHP>{title}</PHP></title>` で受けることもできるが、PHPタグを受け渡す場合のフォーマットは統一しておいた方がよいと思われる。

PHP専用のコンポーネントであればテンプレートリテラル型で定義してもいいかも知れない。

```typescript
interface Props {
  foo: `<?php ${string} ?>`;
} 
```

## 拡張子の書き換え

ビルドされたHTMLのうちPHPとして使用するものを `astro:build:done` フック内で判別して拡張子を書き換える。
判別方法はパスでも二重拡張子でも決め打ちでもなんでも構わない。

```ts:astro.config.ts
import fs from 'node:fs';
import fg from 'fast-glob';

export default defineConfig({
  build: {
    format: 'preserve',
  },
  integrations: [
    {
      name: 'html2php', // 適当でよい
      hooks: {
        'astro:build:done': async ({ dir }) => {
          // 出力ディレクトリ内のHTMLファイルをリストアップ
          const files = await fg.glob(`${dir.pathname}/**/*.html`);
          await Promise.all(
            files.map(async (file) => {
              // 例1: ファイルパスに `wp-content` が含まれていたら拡張子をPHPに書き換え
              if (file.includes('wp-content')) {
                await fs.promises.rename(file, file.replace('.html', '.php'));
              }
              // 例2: 二重拡張子 `.php.html` なら拡張子をPHPに書き換え
              if (file.endsWith('.php.html')) {
                await fs.promises.rename(file, file.replace('.php.html', '.php'));
              }
            }),
          );
        },
      },
    },
  ],
});
```

- 参照: [astro:build:done (Astro Docs)](https://docs.astro.build/en/reference/integrations-reference/#astrobuilddone)
