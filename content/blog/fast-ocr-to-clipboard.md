+++
title = "Fast OCR to clipboard"
date = 2023-01-21
draft = false

[taxonomies]
categories = ["Tech"]
tags = ["tools", "linux"]

[extra]
toc = true
copy = true
math = false
mermaid = false
+++

# Just give me the text

I wanted to share a surprisingly elegant solution I found to copying text into
the clipboard where for one reason or another it's otherwise not possible.

```bash
flameshot gui --raw \
  | tesseract stdin stdout -l eng \
  | xclip -in -selection clipboard
```

<!-- more -->

- [flameshot](https://github.com/flameshot-org/flameshot) ::  Powerful yet simple to use screenshot software
- [tesseract](https://github.com/tesseract-ocr/tesseract) :: Tesseract Open Source OCR Engine
- [xclip](https://github.com/astrand/xclip) :: Command line interface to the X11 clipboard

If you're using [Wayland](https://wayland.freedesktop.org/), substitute `xclip`
for `wl-copy`:

```bash
flameshot gui --raw \
  | tesseract stdin stdout -l eng \
  | wl-copy
```

- [wl-clipboard](https://github.com/bugaevc/wl-clipboard) :: Command-line copy/paste utilities for Wayland

I didn't come up with this incantation, rather found it after a lot of searching
for a tool that solves this problem.

The `flameshot gui` launches a GUI that allows you to select a region of the
screen, and then press `enter`. The image output is piped to `tesseract` which
does the OCR and in turn pipes it to `xclip` or `wl-copy` which copies it to the
clipboard.

Note that `tesseract` has a lot of options, and you can specify the languages.

>That beautifully demonstrates the power of the unix tools philosophy.
>
>Flameshot knows nothing of OCR, tesseract has no gui and doesn't need it.
>
> Do one thing and do it well. — u/Dee_Jiensai [^1]

# Flameshot Alternatives

- [maim](https://github.com/naelstrof/maim) :: (make image) takes screenshots of your desktop.
- [grim](https://github.com/emersion/grim) :: Grab images from a Wayland compositor

Some Reddit users suggested these alternatives to flameshot:

```bash
grim -g "$(slurp)" -
```

```bash
maim -s
```

I haven't tried `grim` because I'm not using Wayland yet, but `maim` is a great!
One advantage of `maim` in this use case is that it doesn't require pressing
`Enter`.

# Discussion

~~Check out the [r/linux](https://www.reddit.com/r/linux/comments/10icyjo/fast_ocr_to_clipboard/) discussion on this blog post.~~

Reddit is dead. Check this out instead: <https://programming.dev/post/81722>

Thanks everyone for the suggestions! I have updated this post with them in mind.
You can see the diff
[here](https://github.com/willbush/blog/commit/145becf969d64074c2761aa7669cf57e96c7c8f8)
if you want to see what changed.

---

[^1]: The above is [a
    quote](https://www.reddit.com/r/linux/comments/10icyjo/fast_ocr_to_clipboard/j5fdvnp/)
    from a user on Reddit in the discussion of this blog post referring to the
    [Unix Philosophy](https://en.wikipedia.org/wiki/Unix_philosophy).
