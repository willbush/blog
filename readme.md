# Blog

A personal site and blog using the [zola](https://www.getzola.org/) static site
generator and wonderful [serene theme](https://github.com/isunjn/serene).

## Run locally

Install https://www.getzola.org/ then:

```sh
git clone https://github.com/willbush/blog.git --recurse-submodules
cd blog
zola serve
```

## Deployed via direct upload

I find clouldflare build and zola support janky so I use direct upload via CI:

https://developers.cloudflare.com/pages/how-to/use-direct-upload-with-continuous-integration/
