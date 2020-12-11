# Use `next/image` on Netlify

This repo is a proof of concept demonstrating how to use the `next/image` functionality on other platforms.

This implementation uses [Netlify Functions](https://www.netlify.com/products/functions/?utm_source=github&utm_medium=next-image-jl&utm_campaign=devex) to process images on the fly by setting up a rewrite rule to handle the [special endpoint `next/image` sends requests to](https://github.com/vercel/next.js/blob/8340e6d34562ad575293b5699023144fc47831d2/packages/next/client/image.tsx#L480).

<details>
  <summary><strong>Expand for additional details about how `next/image` works under the hood</strong></summary>

## Images are sent to a special endpoint for processing

Let's assume the following `next/image` setup:

```jsx
import Image from 'next/image';

export default function MyImage() {
  return (
    <Image
      src="/boop.jpg"
      alt="what a great boop"
      width="1368"
      height="1044"
      layout="responsive"
    />
  );
}
```

Next will render this using the following markup (formatting added for legibility):

```html
<div style="display: block; overflow: hidden; position: relative; box-sizing: border-box; margin: 0px;">
  <div style="display: block; box-sizing: border-box; padding-top: 76.3158%;"></div>
  <img
    alt="what a great boop."
    src="/_next/image?url=%2Fboop.jpg&w=3840&q=75" 
    decoding="async"
    sizes="(max-width: 640px) 640px, (max-width: 750px) 750px, (max-width: 828px) 828px, (max-width: 1080px) 1080px, (max-width: 1200px) 1200px, (max-width: 1920px) 1920px, (max-width: 2048px) 2048px, 3840px"
    srcset="/_next/image?url=%2Fboop.jpg&w=640&q=75 640w, 
            /_next/image?url=%2Fboop.jpg&w=750&q=75 750w, 
            /_next/image?url=%2Fboop.jpg&w=828&q=75 828w, 
            /_next/image?url=%2Fboop.jpg&w=1080&q=75 1080w, 
            /_next/image?url=%2Fboop.jpg&w=1200&q=75 1200w, 
            /_next/image?url=%2Fboop.jpg&w=1920&q=75 1920w, 
            /_next/image?url=%2Fboop.jpg&w=2048&q=75 2048w, 
            /_next/image?url=%2Fboop.jpg&w=3840&q=75 3840w" 
    style="visibility: visible; position: absolute; inset: 0px; box-sizing: border-box; padding: 0px; border: none; margin: auto; display: block; width: 0px; height: 0px; min-width: 100%; max-width: 100%; min-height: 100%; max-height: 100%;"
  />
</div>
```

The image URL was changed from `/boop.jpg` to `/_next/image?url=%2Fboop.jpg&w=750&q=75`.

Let's break down what this does:

```
/_next/image?url=%2Fboop.jpg&w=750&q=75
^^endpoint^^     ^^^image^^^ ^^config^^ 
```

- `/_next/image` — this is an endpoint where the image will be sent for processing
- `?url=/boop.jpg` — where the endpoint should load the image from
- `&w=750` — resize the image to 750px wide
- `&q=75` — resample the image at 75% quality to reduce the file size

Because this endpoint results in a unique URL, the result can be cached to ensure the image processing is only performed once.
</details>

## How this repo adds `next/image` support for other platforms

To provide the `next/image` approach on any platform, including fully static exports, we're doing two things here:

### 1. Create a serverless function to act as the image processing endpoint

The code in [functions/image.js](./functions/image.js) loads the image and processes it using [Jimp](https://www.npmjs.com/package/jimp). The same `url`, `w`, and `q` parameters are supported for compatibility with the `/_next/image` endpoint API.

To test this, you can call the function directly like so:

```
https://next-image-on.netlify.app/.netlify/functions/image?url=/jason-rogers.jpg&w=400&q=75
```

This works with external images as well:

```
https://next-image-on.netlify.app/.netlify/functions/image?url=https://lengstorf.com/images/jason-lengstorf.jpg&w=400&q=75
```

### 2. Use a rewrite rule to send `/_next/image` to the serverless function

To tell `next/image` to use our serverless function, we _could_ mess around with the [configuration](https://nextjs.org/docs/basic-features/image-optimization#configuration). However, since we already have a `netlify.toml`, we're going to use a [rewrite rule](https://docs.netlify.com/routing/redirects/rewrites-proxies/?utm_source=github&utm_medium=next-image-jl&utm_campaign=devex#limitations) to rewrite all traffic from `/_next/image` to `/.netlify/functions/image`.

```toml
[[redirects]]
  from = "/_next/image*"
  query = { url = ":url", w = ":width", q = ":quality" }
  to = "/.netlify/functions/image?url=:url&w=:width&q=:quality"
  status = 200
```

This allows us to use `next/image` without any modifications to the Next.js config, and the images will Just Work™.

As an added bonus, this means we can update the rewrite to use something like [Cloudinary](https://cloudinary.com/?ap=lwj) by changing one line in the rewrite rule:

```diff
  [[redirects]]
    from = "/_next/image*"
    query = { url = ":url", w = ":width", q = ":quality" }
-   to = "/.netlify/functions/image?url=:url&w=:width&q=:quality"
+   to = "https://res.cloudinary.com/jlengstorf/image/fetch/w_:width,q_auto,f_auto/https://next-image-on.netlify.app/:url"
    status = 200
```
