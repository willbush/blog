+++
title = "Fast OCR to clipboard"
description = "An example of a GUI following Unix Philosophy"
date = 2023-01-21

[taxonomies]
categories = ["Tech"]
tags = ["tools", "linux"]

[extra]
toc = false
cc_license = false
+++

# Just give me the text

I wanted to share a surprisingly elegant solution I found to copying text into
the clipboard where for one reason or another it's otherwise not possible.

```bash
flameshot gui --raw | \
  tesseract stdin stdout | \
  xclip -in -selection clipboard
```

- [flameshot](https://github.com/flameshot-org/flameshot) ::  Powerful yet simple to use screenshot software
- [tesseract](https://github.com/tesseract-ocr/tesseract) :: Tesseract Open Source OCR Engine
- [xclip](https://github.com/astrand/xclip) :: Command line interface to the X11 clipboard

I didn't come up with this incantation, rather found it after a lot of searching
for a tool that solves this problem.

The `flameshot gui` launches a GUI that allows you to select a region of the
screen, and then press enter. The image output is piped to `tesseract` which
does the OCR and in turn pipes it to `xclip` which copies it to the clipboard.

I think this is the first time I've seen a GUI tool that follows the [Unix
Philosophy](https://en.wikipedia.org/wiki/Unix_philosophy).

I personally alias this to `ft` in my NixOS (btw) config [like
so](https://github.com/willbush/system/blob/cd423aa52823a16f9486374b572d69296b8d16cd/users/profiles/programs.nix#L54).
