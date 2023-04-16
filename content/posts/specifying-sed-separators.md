---
title: "Custom sed Separators"
date: 2023-01-29T13:32:24-04:00
draft: false
tags: [til]
categories: [programming]
---

When using `sed` for a substitution I was under the impression that `/` had to be used as a separator for the segments.

```bash
sed 's/pattern/replacement/options'
```

Turns out you can use whatever separator you want. This is incredibly handy if the pattern you are substituting contains
slashes - such as urls. By selecting a different separator all of the escaping can be avoided.

**Without custom separator - lots of escaping**
```bash
sed 's/\/posts\/specifying-sed-separators/\/posts\/custom-sed-separators/'
```

**With custom separator - more clear**
```bash
sed 's|/posts/specifying-sed-separators|/posts/custom-sed-separators|'
```

Sed is really good at figuring out what separator you are using and working with it. It's worth calling out the
community tends to use `#` as a separator when a custom one is needed. If you use more exotic separators like I
did above it could cause confusion.

```bash
sed 's#/posts/specifying-sed-separators#/posts/custom-sed-separators#'
```
